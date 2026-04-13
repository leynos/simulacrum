# Architectural decision record 001: GitHub mock capability prioritisation

## Status

Accepted.

## Date

2026-04-13.

## Context and problem statement

`@simulacrum/github-api-simulator` has a roadmap for improving scriptability,
but the original ordering was too collaboration-centric. It prioritised
repository issues, pull requests, reviews, checks, mergeability, and webhooks
in a generic sequence. That shape does not match the current consumer
portfolio.

The portfolio review points in a different direction:

- Concordat is the primary architecture driver. Its value sits in repository
  governance and control-plane automation: repository settings, branch
  protection, team membership, and team-repository permissions.
- Ghillie is a read-heavy GraphQL consumer. It depends more on repository refs,
  commit history, pull requests, and issues than on write-heavy collaboration
  flows.
- shared-actions needs a narrow but high-value slice around pull request state,
  mergeability, checks, auto-merge, and release-tag lookup.
- Nile Valley should have limited influence on prioritisation because its main
  workflows rely more on plain git operations and GitHub Actions file semantics
  than on broad REST or GraphQL parity.

The simulator's current implementation also makes some early foundation work
mandatory. Repositories and branches are keyed too loosely, `/user` resolves to
the first stored user, GraphQL `viewer` is tied to a fixed actor shortcut, and
the GraphQL layer has only a handful of explicit query resolvers with no
meaningful mutation coverage. Those shortcuts are acceptable for thin stubs,
but they are poor footing for scriptable admin or mergeability behaviour.

## Decision drivers

- Maximise value for the current consumer portfolio, not for a hypothetical
  average GitHub client.
- Preserve the vertical-slice approach so that each tranche delivers complete,
  testable behaviour.
- Put a shared actor-aware state model underneath REST and GraphQL before
  expanding parity breadth.
- Prioritise administration and governance surfaces ahead of rich collaboration
  surfaces when Concordat depends on them directly.
- Deliver GraphQL read parity early where Ghillie and shared-actions already
  depend on it.
- Deliver mergeability and auto-merge ahead of deep review-thread fidelity.
- Avoid spending early effort on webhook breadth, Projects v2, or code-scanning
  parity until the core shared substrate is credible.

## Options considered

### Option A: keep the original collaboration-first ordering

The existing roadmap started with a broad collaboration substrate, then moved
through issues, pull requests, reviews, checks, and only later reached
app-facing integration and governance-adjacent work.

Benefits:

- The sequence resembles the public GitHub collaboration narrative.
- It groups related collaboration behaviours together.

Costs:

- Concordat's control-plane needs land too late.
- Ghillie's GraphQL read model waits behind lower-leverage issue mutation work.
- shared-actions does not get mergeability and auto-merge soon enough.
- The simulator risks delivering polished collaboration demos before it can
  drive the portfolio's core governance and automation workflows.

### Option B: reorder the roadmap around consumer-shaped slices

The revised ordering keeps the vertical-slice philosophy but changes the queue:
foundations first, then GraphQL reads, then pull request mutations, then
administration, then checks and mergeability, then review state, then review
threads, then deeper issue parity, then capability contracts, then webhooks,
then later governance extras.

Benefits:

- Concordat gets the administration spine it needs earlier.
- Ghillie gets useful GraphQL reads earlier.
- shared-actions gets mergeability, auto-merge, and release-tag lookup earlier.
- The first delivered slices reflect the actual products consuming the
  simulator today.

Costs:

- Full issue discussion parity lands later than in the previous roadmap.
- Webhook breadth is deliberately delayed.
- Some later governance extras, such as Projects v2 and code scanning, remain
  outside the early queue even though they matter to Concordat.

## Decision outcome

Option B is adopted.

The roadmap should be reordered as follows:

1. Essential foundations only:
   `1.1.1`, a minimal `1.1.2`, `1.2.1`, `1.2.3`, `1.3.1`, and `1.3.3`.
1. A read-heavy GraphQL tranche:
   `2.1.1` plus the read-side parts of `3.1.2` and issue visibility needed for
   current consumers.
1. Pull request mutations and timeline state:
   `3.2.1` and `3.2.3`.
1. A new Concordat administration slice:
   repository settings, branch protection, teams, membership, and
   team-repository permissions.
1. Statuses, checks, mergeability, auto-merge, and early release-tag lookup:
   `5.1.1`, `5.1.2`, `5.2.1`, `5.2.2`, GraphQL auto-merge state, the
   `enablePullRequestAutoMerge` mutation, and
   `GET /repos/{owner}/{repo}/releases/tags/{tag}`.
1. Review state before review threads:
   `4.1.x` ahead of `4.2.x`.
1. Remaining issue mutation and timeline work:
   the rest of `2.2` and `2.3`.
1. Capability matrix, fixture recipes, and scenario suites.
1. Webhooks and delivery inspection.
1. Later governance extras such as Projects v2 and code scanning.

## Goals and non-goals

Goals:

- Build a Concordat-shaped administration spine.
- Build a Ghillie-shaped GraphQL read model.
- Build a shared-actions-shaped mergeability and auto-merge slice.
- Keep REST and GraphQL backed by shared entities, actor resolution, and domain
  events.

Non-goals:

- Achieve broad GitHub collaboration parity before control-plane behaviour is
  credible.
- Build a deep permission lattice before branch protection and team behaviour
  require it.
- Prioritise webhook breadth ahead of polling and direct-read consumers.
- Treat Projects v2 or code-scanning parity as immediate blockers for the
  shared substrate.

## Migration plan

- Update `packages/github-api/docs/roadmap.md` to reflect the new tranche
  order.
- Keep the existing task identifiers as reference labels where useful, but make
  delivery order explicit in the roadmap rather than implied by the old phase
  numbering.
- Introduce a new administration slice for repository settings, branch
  protection, teams, and permissions.
- Add explicit roadmap tasks for GraphQL auto-merge support and release-tag
  lookup because those needs are not named clearly enough in the previous
  version.

## Known risks and limitations

- Deferring the full permission lattice may hide some authorisation edge cases
  until the administration tranche deepens.
- Prioritising the current GraphQL query shapes may optimise the model around
  today's consumers more than tomorrow's ones.
- Delaying webhook breadth means some GitHub App integration tests will still
  require polling-oriented fixtures in the near term.
- Projects v2 and code-scanning parity remain open prioritisation questions
  once the shared substrate is in place.

## Architectural rationale

The simulator should first become credible where the portfolio already depends
on it. That means an administration spine for Concordat, a practical GraphQL
read model for Ghillie, and a mergeability slice for shared-actions. Richer
collaboration behaviour still matters, but it should follow the surfaces that
shape repository policy, branch safety, and automation decisions. This keeps
the vertical-slice approach intact while changing the queue to reflect actual
consumer leverage.
