# 0003: kcp staging repositories ("monorepo") proposal

GitHub issue: https://github.com/kcp-dev/kcp/issues/3375

## Background and motivation

A module is a collection of Go packages stored in a file tree with a `go.mod` file at its root. The `go.mod` file
defines the module’s module path, which is also the import path used for the root directory, and its dependency
requirements, which are the other modules needed for a successful build. Each dependency requirement is written
as a module path and a specific semantic version [[source]](https://go.dev/blog/using-go-modules#introduction).

A module has some degree of coupling in relation to other modules within the project by depending or being
a dependency of other modules. A well-encapsulated module is one that has the least amount of coupling with respect
to others in the project. Such a module is then a good candidate for being hosted in its own separate VCS repository,
gaining self-governance, its own issue tracker, PRs, CI, release cycle freedom, and many more.

On the other hand, many inter-dependencies between modules may give a hint that hosting them together in a single
repository may be preferable. In these cases, change in one module often has to be reflected in changing the others,
even if it means changing just the tag of that dependency in `go.mod`. Hosting in a shared repository means we lose
some of the freedoms separate repos give us, but this can be partially worked around. We however gain a much more
streamlined development experience (single PR across more modules), reviews (see the whole picture) and releases
(things that depend on each other may be released together).

In contrast, hosting inter-dependent modules in separate repositories duplicates a lot of the maintenance work
(keeping `go.mod` files in sync across the repositories, tagging releases, etc.), and over-arching changes require
a lot of effort (e.g. a change in `kcp-dev/code-generator` means one has to open a subsequent PR in `kcp-dev/client-go`,
and once that is merged, the developer still needs to open a PR for `kcp-dev/code-generator/example`). Context of kcp
makes this extra hard during Kubernetes rebases, where doing a review and testing of the whole change is almost
impossible without a lot of manual effort due to the repository split.

At the time of writing this document (May 2025), there are thirteen maintained golang repositories in
`github.com/kcp-dev` (last commit not older than two months, or important repos like `kcp-dev/logicalcluster`),
with varying degrees of coupling between them. Some of the modules in these repositories may have been misclassified
as being fit to be hosted in a separate repository, others may have accumulated changes over time that make them fit to
be moved into another repository.

In this proposal we’d like to move development of a subset of modules found across the `kcp-dev` organization into the
`kcp-dev/kcp` repository, gaining benefits described above.

## Goals and non-goals

This proposal focuses on the technical side of how to perform module mobility to opt-in or -out of the monorepo
structure. This is not an analysis of which modules within the kcp-dev organization are fit for mobility. Having said
that, there are clear candidates for that already:

- kcp-dev/apimachinery
- kcp-dev/client-go
- kcp-dev/code-generator

Goals:

- Put in place tooling needed for module management in a monorepo structure.
- Design a workflow for moving module(s) from a repository within the organization into kcp-dev/kcp, without breaking its import paths.
- Design a workflow for moving a moved-in module back into its original repository, without breaking its import paths.
  This reverts the action from the point above.

The tooling itself already exists and is in use by Kubernetes:
- scripts in [kubernetes/kubernetes/hack](https://github.com/kubernetes/kubernetes/tree/master/hack),
- GitHub bot [kubernetes/publishing-bot](https://github.com/kubernetes/publishing-bot).

Non-goals:

- Define a complete set of modules to be moved into kcp-dev/kcp.
- Modify code in modules to be fit for monorepo mobility.

## Proposal

We propose to use the staging model that Kubernetes implemented and uses. The authoritative version of the code lives
in the `staging/` directory within the monorepo (e.g. `kcp-dev/kcp` in our case), and the respective split repositories
host only a read-only copy of it to retain the import paths.

The publishing-bot is a project maintained by Kubernetes that has a goal to enable this pattern. publishing-bot is
implemented in Go and Bash, and it utilises different capabilities of Git under the hood. A detailed overview of how
publishing-bot works is available in [the project’s repository](https://github.com/kubernetes/publishing-bot/blob/57254781ddfd09e63aa592eb01a0dff182c4224f/artifacts/scripts/util.sh#L24-L124).

The publishing-bot publishes the code in `kcp-dev/kcp/staging` to their own repositories. It guarantees that the main
branches of the published repositories are compatible, i.e., if a user runs go get on a published repository in a clean
`GOPATH`, the repo is guaranteed to work.

It pulls the latest `kcp-dev/kcp` changes and runs `git filter-branch` to distill the commits that affect a staging
repository. Then it cherry-picks merged PRs with their commits to the target staging repository. It records the commit
hash (SHA1) of the original commit in the commit message of the cherry-picked commit (`Kcp-sha: <sha1>` line). This
recorded hash is used as a reference point for syncing (i.e. sync is done between that commit and the latest commit).

The publishing-bot is also responsible to update the `go.mod` and `go.sum`, and the `vendor/` directory for the target
repositories.

Finally, the publishing-bot is responsible for creating tags in the staging repositories based on tags in the monorepo.

We would run a slightly modified version of publishing-bot on our infrastructure to accommodate differences between
Kubernetes and kcp. publishing-bot runs periodically, every couple of hours. This means that some commits and tags
might be missing for a brief time between two runs, however, we don’t consider this a problem.

The staging directory in `kcp-dev/kcp` contains Go modules with all the elements as they would have in a dedicated
repository (including `go.mod` and `go.sum`), but without the .git directory. The staging directory hierarchy looks
like this:

```
kcp-dev/kcp
└── staging
    ├── publishing
    │   └── rules.yaml ← contains config and rules for p-b
    └── src
        └── github.com
            └── kcp-dev
                ├── apimachinery
                    └── go.mod
                    ├── go.sum
                    └── …
                ├── client-go
                └── …
```

Commits and tags are synchronized only for branches that are defined in the rules file (rules.yaml).
The rules file contains:

- Mappings between branches in the monorepo and branches in the staging repos
- Go version for each branch (Go is used by publishing-bot for maintaining go.mod and go.sum)
- Additional metadata (e.g. dependencies, what files to remove…)

## Implementation and testing

### Modifying publishing-bot

The publishing-bot needs to be modified because it has some specific to the Kubernetes project. The list of required
changes is as follows:

- Configuration files (all included configuration files are Kubernetes specific, below are configuration files adjusted
  for kcp)
- Change the default StorageClass from `ssd` to the desired StorageClass (`Makefile`, `artifacts/manifests/pvc.yaml`)
- Change `GIT_COMMITTER_NAME` and `GIT_COMMITTER_EMAIL` to the kcp-owned GitHub account
- Some minor changes to the scripts and bug fixes are required
  - Use the correct remote for some Git commands (bug in publishing-bot)
  - Ignore tags not starting with `v` (e.g. `sdk/`)
  - Value of the `--mapping-output-file` flag for `sync-tags` needs to be adjusted to write mapping files one level
    higher (because we have `github.com/kcp-dev/kcp` (three levels) instead of `k8s.io/kubernetes` (two levels))

Most of these changes could eventually be backported to the original publishing-bot repository (aside from the
configuration files), but this is out of scope for this proposal.

We would need to push a custom image of publishing-bot with these changes. We can use the Dockerfile coming with
publishing-bot to build the image, and then push it to the GitHub Image Registry using either GitHub Actions or Prow.
The image reference needs to be added to the publishing-bot repository config (see configuration files used below).

### Preparing the monorepo (`kcp-dev/kcp`)

We need to prepare the monorepo to contain the staging repositories. This is as simple as creating the following
directory tree:

- `staging/publishing` - this directory contains the rules file
- `staging/src/github.com/kcp-dev` - this directory contains the staging repositories

### Staging a new repository

A new repository in this context assumes that we’re creating a new repository from scratch that we want to stage from
the monorepo. **publishing-bot, in fact, only supports this case.**

1. Create a new staging repository on GitHub
1. Push an empty “Initial commit” to the new repository
1. Create the appropriate directory in the monorepo (e.g. `kcp-dev/kcp/staging/src/github.com/kcp-dev/<new-repo-name>`)
   and put the desired code into that directory
1. Update the publishing-bot rules to include this staging repository
1. Push changes to the monorepo and run publishing-bot
1. Ensure the desired code is pushed to the staging repository after publishing-bot finishes successfully
1. Ensure the tags are properly synchronized to the staging repository

publishing-bot synchronizes all tags from the monorepo to the target staging repository. When a new staging repository
is created, all tags existing in the monorepo at that time will be created in the staging repository, each pointing to
the “Initial commit” as their target commit. This is only for the initial synchronization, all future tags will be
correctly created with the latest commit as their target commit.

### Staging an existing repository

Staging an existing repository is not supported by publishing-bot. publishing-bot can detect and bootstrap a new
repository as described in the previous section, but it doesn’t know how to handle an existing repository, i.e. a
branch that has pre-existing commits on it.

Note: empty branch means a branch that only has an empty commit on it. We’ll usually refer to this commit as the
Initial commit.

There are two mitigations for this problem:

- Create a new empty branch in the existing staging repository, while renaming and keeping the old branch for the
  commit history.
  - A new empty branch can be connected with the monorepo as described in the previous section.
  - The commit history is still there as we’re keeping the old branch (just under a different name). It should still be
    possible to `go get` (fetch) the old commits.
  - The biggest con about this approach is that the new branch doesn’t have history, making it harder to navigate
    historical changes. Some users might perceive this as a breaking change depending on their workflows.
- Rebuild the existing branch.
  - This is an extension of the previous approach. We would create a new empty branch in the existing staging
    repository and point publishing-bot to that new branch. Once publishing-bot pushes the code to that new branch, we
    would utilize different git options to rebuild the existing/old branch to contain old commits (i.e. the history)
    and the commit pushed by publishing-bot to the new branch, while ensuring the work tree hash of the old/rebuilt
    branch matches the work tree hash of the new branch.
  - This way we keep the history, and we get a branch that’s acceptable for publishing-bot. The biggest con about this
    approach is that some additional effort is required initially, however, this is not a factor because the process
    will be well documented.

It’s important to note that due to how publishing-bot is implemented, any mitigation must satisfy:

- Initially, we must start from the empty branch because publishing-bot cannot handle existing branches.
- Any attempt to rebuild the existing/old branch must satisfy:
  - The worktree hash of the rebuilt branch must match the worktree hash of the branch bootstrapped by publishing-bot
    (the worktree hash can be obtained using `git rev-parse ${BRANCH_NAME}^{tree}`)
    - This ensures that the contents of the rebuilt branch is byte to byte same as the contents of the branch bootstrapped
    by publishing-bot. This is especially important as publishing-bot might modify contents inside the staging
    repositories (especially `go.mod` and `go.sum` files).
  - The latest commit on the rebuilt branch must contain the appropriate `Kcp-commit` reference in its commit message.
    - This ensures that publishing-bot can pick up the rebuilt branch and connect it with the appropriate commit in
      the monorepo.

There’s an example solution for rebuilding the branch included in the Appendix II of this document.

## Unlinking or archiving a staged repository

In case we want to unlink a staged repository (i.e. make it a standalone repository) or archive it, it’s enough to
simply remove all rules for that repository from the publishing-bot rules in the monorepo.

## Writing publishing-bot rules

A detailed document on how to write publishing-bot rules will be provided if this proposal gets accepted.
The [publishing-bot rules for Kubernetes](https://github.com/kubernetes/kubernetes/blob/master/staging/publishing/rules.yaml)
can be used as an example to see most of the options that publishing-bot offers.

## Developer experience

To ensure the smooth developer experience, the root `go.mod` file in the monorepo shall be extended with the
appropriate replace directives for each staging repo, e.g.

```go
replace (
    github.com/kcp-dev/client-go => ./staging/src/github.com/kcp-dev/client-go
)
```

Additionally, we should consider adopting the [Go workspaces](https://go.dev/blog/get-familiar-with-workspaces) as they
make it easier to work with multiple Go modules. Additional considerations are needed for this, as well as, analysis of
benefits and drawbacks, so this is out of the scope of this proposal. See the Kubernetes 
[KEP-4402](https://github.com/kubernetes/enhancements/blob/5c1539d9616021d25a003ba68b2793ad5cdb31cf/keps/sig-architecture/4402-go-workspaces/README.md) for more details.

All PRs against the staging repositories (e.g. `kcp-dev/client-go`) must be rejected and contributors must be pointed
to the monorepo. Accepting any PR in the staging repository can lead to serious problems with publishing-bot and must
be avoided at any cost. This shall be mentioned in the PR template for the staging repository and `CONTRIBUTING.md`.

## Changes to the release process

Currently, we’re manually tagging releases in all repositories that are candidates for becoming the staging
repositories. Once they become staging repositories, the release process will be handled via the monorepo.
Any new tag from the monorepo will be automatically created in all staging repositories on the next publishing-bot run.

## Handling the existing tags

Some of the existing repositories that are candidates for becoming staging repositories in the `kcp-dev` organization
have tags. Those tags might in a way conflict with tags pushed by publishing-bot. For example, publishing-bot will push
tags like v0.28.0 (that match the kcp version/tag), but some repositories have tags like v2.x.y. This should be handled
on a case-by-case basis.

Alternatively, we can disable tags synchronization in publishing-bot, however, this makes the release process much more complicated and time consuming.

## Permissions required for implementation

The admin level permissions will be needed for implementation due to required force pushes and branch creations in the
staging repositories.

## User experience

publishing-bot ensures that we keep the same import paths for the staging repositories as before (e.g. for client-go,
the import path will remain `github.com/kcp-dev/client-go` as it is right now). This is because the relevant code is
replicated from the monorepo to the concrete staging repositories, so users are always pulling modules from these
repositories instead of the monorepo.

What might change is the versioning schema, as described in “Changes to the release process” and “Handling the existing
tags” sections. This shall be communicated to the users via the official mailing lists ahead of implementing
publishing-bot.

## Testing

Any implementation of this proposal shall be tested in an organization different from `kcp-dev`. Only after a
successful testing phase, the implementation might be applied to the `kcp-dev` organization.

The easiest way to test any implementation is to fork all relevant repositories from `kcp-dev` to that new testing
repository (let’s name it `kcp-nightly` for the purposes of this proposal). We would configure publishing-bot to use
`kcp-nightly` as the source and target organization, and then run publishing-bot and all required one-time commands
manually on a local machine.

It would be beneficial to have a GitHub organization different from `kcp-dev`, owned by the kcp maintainers, that can
be used both for the initial testing and for testing any future extensive (in scope) changes after the implementation.
Kubernetes uses the [`kubernetes-nightly`](https://github.com/kubernetes-nightly) organization for these purposes.

## Alternatives considered

At this time, we didn’t consider alternatives because our use case is not common and we’re not aware of other tools
that allow this pattern.

## Appendix I: configuration files used

publishing-bot configuration (publisher-config ConfigMap):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: publisher-config
data:
  config: |
    source-org: <org-name>
    source-repo: kcp
    target-org: <org-name>
    rules-file: https://raw.githubusercontent.com/<org-name>/kcp/refs/heads/main/staging/publishing/rules.yaml
    github-issue: <github-issue-number-in-org-name/kcp>
    dry-run: false
    git-default-branch: main
```

publishing-bot rules ConfigMap (empty because we’re reading rules from GitHub):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: publisher-rules
data:
  config: |
```

publishing-bot repository config:

```ini
DOCKER_REPO = <repo>/publishing-bot
NAMESPACE = publishing-bot
SCHEDULE = * */4 * * *
INTERVAL = 14400
CPU_LIMITS = 2
CPU_REQUESTS = 300m
MEMORY_REQUESTS = 2Gi
MEMORY_LIMITS = 2Gi
```

## Appendix II: commands and notes from the initial testing

An initial testing has been performed while writing this proposal to test out different possibilities. Below are
commands and notes from these testing attempts. A separate GitHub organization was used for testing purposes, which is
why steps and commands like forking are included.

Prepare an existing repository for staging:

1. Create a fork of the staging repository (e.g. `kcp-dev/client-go`) and clone the repo locally
1. Checkout the desired branch (e.g. `main`) and put its name to a variable: `export BRANCH_NAME="<branch-name>"`
1. Copy everything from the staging repository to `github.com/<testing-org>/kcp/staging/src/github.com/kcp-dev/<repository-name>`
1. Remove the `.git` directory from `github.com/<testing-org>/kcp/staging/src/github.com/kcp-dev/<repository-name>`
1. Switch to the `<testing-org>/kcp` repository locally, commit adding the staging repository, and push it to the branch
1. Switch back to the staging repository
1. Create an empty branch: `git switch --orphan new-${BRANCH_NAME}`
1. Create an empty commit on the new branch: `git commit --allow-empty -m "Initial commit"`
1. Push the branch to the fork: `git push origin new-${BRANCH_NAME}`

Prepare the initial publishing-bot rules:

Put and push the following publishing-bot rules in `<testing-org>/kcp/staging/publishing/rules.yaml`:

```yaml
rules:
- destination: <repository-name>
  branches:
  - name: new-<branch-name-in-staging-repo>
    source:
      branch: <branch-name-in-kcp>
      dirs:
      - staging/src/github.com/kcp-dev/<repository-name>
  library: true
recursive-delete-patterns:
- '*/.gitattributes'
default-go-version: 1.24.2
skip-tags: true
```

(note `skip-tags: true`, this is expected, tags should be synced only after successful connection between the monorepo
and the staging repository)

Run publishing-bot:

At this point, run the publishing-bot to let it upload the initial state to the new branch in the staging repository.

Restructure the old branch and history:

**WARNING: THIS ASSUMES THAT THERE'S ONLY ONE COMMIT FROM PUBLISHING-BOT, OTHERWISE ADJUSTMENTS ARE NEEDED**

1. After the new branch has been updated by publishing-bot, pull it locally
1. Checkout the new branch
1. Get the worktree hash of the new branch: `export new_worktree_hash=$(git rev-parse "new-${BRANCH_NAME}^{tree}")`
1. Get the commit message of the latest commit on the new branch: `export commit_msg=$(git log -1 --pretty=%B)`
1. Get the hash of the HEAD commit on the old branch: `export parent_commit=$(git rev-parse ${BRANCH_NAME})`
1. Create an artificial commit that uses the worktree of the new branch and has HEAD of the old branch as its parent:
   `echo "$commit_msg" | git commit-tree "$new_worktree_hash" -p "$parent_commit"`
1. Checkout the old branch: `git checkout ${BRANCH_NAME}`
1. Reset the branch to the commit outputted by `git commit-tree`: `git reset --hard <hash>`
1. Ensure the latest commit on the branch has `Kcp-commit` in its commit message
1. Force push the branch to the repository: `git push origin -f ${BRANCH_NAME}`
1. Remove the “new” branch: `git branch -D new-${BRANCH_NAME} && git push --delete origin new-${BRANCH_NAME}`

Reconfigure the publishing-bot and pushing tags:

1. Update the publishing-bot rules to change the target branch from `new-<branch-name>` to `<branch-name>`
1. Commit and push changes to the `<testing-repo>/kcp` repo, then run the publishing-bot and ensure it finishes
   successfully
1. Update the publishing-bot rules to remove or disable `skip-tags`, push the changes to the repo, and then run
   publishing-bot
