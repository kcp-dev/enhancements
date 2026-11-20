# LogicalCluster rebalancing

## Overview

A standalone Go CLI that rebalances LogicalClusters across
operator-managed KCP shards. The balancing policy is selected with
`--rebalance-type`; the default `simple` policy balances workspace counts.
It previews a deterministic plan by default and executes migrations only with
`--apply`.

## Motivation

Poor distribution of LogicalClusters (LCs) is not currently addressed, which
limits scalability. Newly added shards remain empty unless new LCs are created.
This proposal is related to the
[Balanced LogicalCluster Scheduling for Multi-Shard epic #333][epic-333].

### Goals

The tool should rebalance LCs across currently available shards to mitigate
overloaded or idle shards. It should support different balancing policies and
consider multiple deployment parameters to find balance, such as etcd database
allocation, CPU and memory utilization, the number of LCs on each shard
and so on.

### Non-Goals

Scheduling new LCs onto shards in a balanced manner is out of scope.

## Proposal

![progam-flow][program-flow]

### SimplePlanner
Count LCs on shards and distribute them so that maximum and minimum number
differ by at most one.

### MILPPlanner
Use *Mixed Integer Linear Programing* to solve a linear system of equations which
reflects the rebalancing objective. Currently the objective minimizes shard load 
deviation from the average load and migration cost. Shard load is calculated by
adding all weights of LCs belonging to the shard. Migration cost as well as the weight
is equal to one for every migration and LC for the time being.

## Implementation
Private repo: https://github.com/simon-fredrich/kcp-rebalancer

### 1. Process lifecycle

[The entry point][main.go] creates a context canceled by Ctrl+C or SIGTERM,
passes command-line arguments and output streams to the CLI, and reports
failures on stderr with a nonzero exit status. Help exits successfully.

### 2. CLI configuration

[The CLI][cli.go] parses flags with a fresh `flag.FlagSet` so
tests and repeated calls do not share global flag state. It validates positional
arguments, the migration timeout, `--rebalance-type`, and repeatable
`--shard-server` HTTPS endpoint overrides before opening connections. The
rebalance type names a planner in the CLI's `planners` registry. The default
policy is `simple`; preview mode remains the default.

### 3. Connect to the shards

[Discovery][discovery.go] loads the KCP kubeconfig and lists
Shards through `/clusters/root`. Each shard uses its advertised `spec.baseURL`,
unless a `--shard-server` override supplies an externally reachable endpoint.
Unknown shard overrides, duplicate or missing shard names, invalid endpoints,
and empty discovery results fail explicitly. The same kubeconfig credentials
and TLS settings are copied to each shard client.

The Go program does not launch subprocesses or manage tunnels.
[The credential script][admin.kubeconfig] can forward
all shards with `-p` and prints the required overrides. Keep it running while
using those endpoints.

The root configuration retains the kubeconfig endpoint with `/clusters/root`
for discovery and migration requests.

### 4. Inventory and planning

[Inventory][inventory.go] reads
`/clusters/*/apis/core.kcp.io/v1alpha1/logicalclusters` sequentially from every
shard; it is not an atomic snapshot. It excludes `root` and `system:*`
clusters, and requires ordinary clusters to have `kcp.io/cluster` and
`kcp.io/path` annotations, a `Ready` phase, and a unique identity across
shards. KCP inventory assigns every workspace a weight of one.

[The planner interface][plan.go] separates policy from
execution. A planner receives the inventory and provides three operations:

- `Plan` proposes moves in execution order.
- `Assess` reports whether a placement satisfies the policy, with reasons if it
  does not, or an error if assessment fails.
- `PrintPlan` prints the policy-specific summary.

Planners work with Go domain types, are deterministic, and do not modify their
input. The [simple planner][simple.go] sorts shard names
and candidate cluster identities, then simulates moves from the fullest shard
to the emptiest until counts differ by at most one. The
[MILP planner][milp.go] minimizes total absolute
deviation of shard load from average plus `MigrationCost` per move, then
prefers fewer moves among solutions with the same objective. A zero migration
cost uses the default of one; negative or non-finite costs are rejected.
Because KCP inventory currently assigns unit weights, MILP load currently
corresponds to workspace count as well.

The MILP policy considers its placement satisfied when its optimizer proposes
no moves. This may leave unequal shard loads when the potential improvement
does not outweigh migration costs.

### 5. Preview or apply

[The runner][runner.go] assesses the current placement,
asks the planner for moves, and rejects the plan if any move cannot be executed:
the workspace is not on the stated source, the destination is unknown or equal
to the source, or a workspace is moved twice. It then prints current and
planned counts or loads, the current assessment, and the proposed moves. In
preview mode it returns without checking or creating migration objects.

With `--apply`, it:

1. Refuses to proceed if a migration in root is not Completed or Failed.
2. Reads inventory again before each move and compares it with expected placement.
3. Creates a migration and waits for completion before submitting the next.
4. Updates expected placement after each completed move.
5. Reads final inventory, verifies that it matches the expected placement, and
   asks the planner to assess it. An unsatisfied assessment or assessment error
   fails execution.

[Migration handling][migration.go] checks for existing
migrations in root and refuses to proceed if any is neither Completed nor
Failed. It creates `LogicalClusterMigration` objects and polls immediately,
then every two seconds. API errors, a Failed phase, timeout, or cancellation
stop execution. An already submitted migration can continue independently of
this process.

## TODO/Questions
- Metrics to base balance decisions on (LogicalCluster metrics not available, Shard metrics are though)
- How are shards discovered best? Currently through `/clusters/root`
- How to list workspaces? Currently through `/clusters/*/apis/core.kcp.io/v1alpha1/logicalclusters`
- Build deployable job or controller beyond manual cli usage

[epic-333]: https://github.com/platform-mesh/backlog/issues/333
[program-flow]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/docs/program-flow.png?raw=true
[main.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/main.go
[cli.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/cli/cli.go
[discovery.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/kcp/discovery.go
[admin.kubeconfig]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/hack/generate-kcp-admin-kubeconfig.sh
[inventory.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/kcp/inventory.go
[plan.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/rebalance/plan.go
[simple.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/rebalance/simple.go
[milp.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/rebalance/milp.go
[runner.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/rebalance/runner.go
[migration.go]: https://github.com/simon-fredrich/kcp-rebalancer/blob/main/internal/kcp/migration.go