# Advanced LogicalCluster Scheduling

## Overview

This proposal aims to improve the balance when scheduling logicalclusters. Currently scheduling is
random, which leaves new shards underutilized.

## Motivation and Context

We want a better distribution of logicalclusters across shards, specifically when a new shard is
added to the system. Currently all scheduling is random, which results in new shards receiving the
same rate of new logicalclusters, leaving them underutilized. Instead, a newly added shard should
overproportionally receive scheduling requests for new logicalclusters to eventually catch up with
older shards, without being overwhelmed. The algorithm must work regardless of installation size
(some have 3 shards, others 100+) and allow multiple parties (FrontProxies) to make scheduling
requests simultaneously on the same pool of shards. We assume that the number of requests for new
workspaces does not severely outnumber the time it takes for a logicalcluster to be ready divided by
the total number of shards. Additionally, FrontProxy should not talk directly with CacheServer (TODO
insert KEP).

### Goals

* Implement a balancing logicalcluster scheduler for workspace requests through FrontProxy
* Allow for the scheduling algorithm to be pluggable for future alternatives
* Start implementation of the Power of Two Choices algorithm (P2C) (for details see proposal below)
  as the first scheduling algorithm
* Minimize traffic between shards for scheduling decisions

### Non-Goals

* Implement cordoning of shards. The current proposal is designed to work with cordoning in the
future (by sending a filtered list to P2C), but implementing it is not within the scope of this
proposal in order to keep it lean and focused
* Taking sizing factors of the shards themselves (e.g. vCPU or RAM) into account. For now we simply
treat each shard as equal in capacity
* Allow users to provide their own scheduling algorithm. While we want to do this eventually, this
  is not the goal of the initial proposal, but will be possible long-term since the algorithm is
  pluggable
* Removing the direct shard-to-shard writes that create and delete a LogicalCluster on the chosen
  shard. This proposal only changes who decides where a LogicalCluster goes, not who executes
  the placement. See the alternatives section for the model that would remove them, but requires
  lots of changes to our existing model.

## Proposal

Add a scheduler component to FrontProxy which handles requests for workspaces. Whenever a new
workspace gets created, it uses the following P2C algorithm to figure out on which shard the
LogicalCluster should be scheduled on:

  1. Draw 2 shards at random from the list of available shards
  2. Compare number of LogicalClusters on both shards
  3. Pick the shard with the lowest number and set the `tenancy.kcp.io/shard` annotation on the Workspace
  4. Create the Workspace object on the parent shard

Afterwards the scheduling process of the LogicalCluster takes place (Note: This part is completely
unchanged by this proposal):

  1. Shard with the new Workspace object reads `tenancy.kcp.io/shard` value and creates the LogicalCluster
     on that shard
  2. Shard with new Workspace object sets spec.cluster and spec.URL on workspace object
  3. Shard with new LogicalCluster replicates the readiness via status to CacheServer
  4. Shard with new Workspace object watches CacheServer and updates Workspace status

The functionality will be made available under a feature flag `--logical-cluster-scheduling="p2c"` in
FrontProxy.

### Why P2C

* Requires us to fetch only two samples regardless of the number of shards
* New shard initially receives 2x the placements compared to existing shards and automatically
converges to the same placement rate over time
* Completely stateless, does not require us to keep any information on past schedulings

## Alternatives Considered

### Always schedule the LogicalCluster on the same shard as its Workspace

This would remove the shard-to-shard write for LogicalCluster creation entirely. However Workspaces
and LogicalClusters could only be scheduled on their parents shard, resulting in single-shard hierarchies,
which defeats the whole purpose of this proposal.

### FrontProxy decides and executes the placement

Instead of only stamping the decision onto the Workspace, FrontProxy would perform the whole
placement: pick a shard via P2C, create the LogicalCluster on that shard directly, and then create
the Workspace on the parent's shard with `spec.cluster`, `spec.URL` and the shard annotation already
set. The parent shard's controller would only track phase through the cache server, and
shard-to-shard writes for creation would disappear, replaced by FrontProxy-to-shard writes, which
is traffic FrontProxy already produces for its index. Deletion would be inverted as well: the owning
shard would garbage-collect its LogicalCluster when the owning Workspace disappears, observed through
the cache server. A variant of this is a pull-based claim model, where shards watch replicated
placement intent and claim Workspaces addressed to them.

This is the direction we think scheduling should eventually move, since it aligns with the principle
that no shard connects to another shard directly. We did not choose it as the initial step because
it changes several general principles at once: FrontProxy becomes a writer of LogicalClusters rather
than a pure router, LogicalCluster ownership moves out of the parent's workspace controller, and
deletion through garbage collection requires Workspaces (or a new intent object) to be replicated to
the cache server, which is a significant increase in replicated volume with its own trade-offs. The
proposal above only changes who makes the scheduling decision and keeps the execution path as it is,
which makes it a small, reversible step that can later be extended into this model.

### FrontProxy connects randomly to one shard, which places the scheduling request on the CacheServer

FrontProxy would forward the workspace request to a randomly chosen shard. That shard would then
write the scheduling request to the CacheServer, from where the other shards would pick it up and
decide among themselves who takes the logicalcluster. While this keeps the idea of FrontProxy not
having to talk to CacheServer directly, it makes the CacheServer a single point of failure for
scheduling.

### FrontProxy connects directly to the CacheServer

Instead of routing through a shard, FrontProxy would write scheduling requests directly to the
CacheServer. This alternative still makes the CacheServer a single point of failure for all
scheduling decisions.

### Shards coordinate scheduling among each other instead of FrontProxy

Shards would talk to each other directly to negotiate where a new logicalcluster should be placed.
This is not in line with the goal of minimizing traffic between shards for scheduling decisions, and
it couples the availability of scheduling to cross-shard connectivity, which we do not want to have
in multi-region setups.

### Weighted algorithm that tracks freshly scheduled logicalclusters

A weighted scheduler would track how many logicalclusters were recently placed on each shard and
factor this into its decisions, for example to avoid overwhelming a shard whose recent placements
have not become ready yet. This was rejected for two reasons. First, it complicates the scheduling
algorithm considerably and requires deep insight into how long it takes a system to schedule a
logicalcluster, a duration which can also vary between installations. Second, such tracking only
becomes effective if the number of requests for new workspaces severely outnumbers the time it takes
for a logicalcluster to become ready divided by the total number of shards - a situation we don't
think happens in real-world applications of kcp.
