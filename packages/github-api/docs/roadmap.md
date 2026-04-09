# GitHub API scriptability roadmap

This roadmap sets out the work needed to make
`@simulacrum/github-api-simulator` fully scriptable for the most commonly used
GitHub workflows. It starts from the current capability gaps described in
`packages/github-api/docs/github-rest-api-audit.md` and
`packages/github-api/docs/github-graphql-api-audit.md`.

The roadmap uses a vertical slice approach. Phase 1 delivers the minimum
foundations required for reliable write behaviour. Every later phase then
delivers a complete, testable collaboration slice rather than another layer of
abstract plumbing.

## Scope and operating principles

- Focus on the GitHub functionality most often used by application developers,
  bots, and internal platform tooling: repositories, issues, pull requests,
  review threads, checks, merges, webhooks, and notifications.
- Prefer shared domain entities and behaviour over one-off endpoint stubs.
- Treat REST and GraphQL as parallel views over the same underlying state.
- Keep each step outcome-driven, measurable, and independently valuable.
- Do not target perfect parity with every GitHub product surface in this
  roadmap. Enterprise administration, packages, Actions orchestration, code
  scanning, and billing remain out of scope unless they are needed to support
  the collaboration slices below.

| Phase | Delivered capability                     | Why it matters                                                                   |
| ----- | ---------------------------------------- | -------------------------------------------------------------------------------- |
| 1     | Shared writeable collaboration model     | Unlocks reliable mutations, identity, and eventing.                              |
| 2     | Repository and issue collaboration       | Delivers the first complete day-to-day workflow.                                 |
| 3     | Pull request lifecycle                   | Makes branch-based change proposals scriptable end to end.                       |
| 4     | Reviews and review threads               | Enables review tooling, approval flows, and threaded discussion.                 |
| 5     | Checks, mergeability, and merge outcomes | Supports branch protection and automation-driven merges.                         |
| 6     | App-facing integration surfaces          | Makes the simulator useful for GitHub App and Backstage-style integration tests. |

_Table 1: Phase-level roadmap summary._

## 1. Core collaboration substrate

This phase is the only foundation-heavy phase. Its purpose is to establish a
shared domain model and mutation framework that later slices can reuse without
rewriting state handling for every feature.

### 1.1. Canonical GitHub identity and storage model

- [ ] 1.1.1. Re-key core store entities by canonical identifiers.
  - Replace repository and branch keying by plain `name` with stable composite
    identifiers such as `owner/name` and `owner/name:ref`.
  - Ensure two repositories with the same name under different owners can
    coexist in the same test fixture.
- [ ] 1.1.2. Introduce first-class collaboration entities.
  - Add store entities for commits, refs, issues, issue comments, pull
    requests, pull request reviews, review comments, review threads, labels,
    assignees, milestones, check suites, check runs, and webhook deliveries.
  - Define relation fields explicitly so REST and GraphQL selectors can derive
    read models without ad hoc joins.
- [ ] 1.1.3. Define immutable identifier and timestamp rules.
  - Standardise id generation, node id generation, timestamps, and URL
    generation for newly created entities.
  - Completion criteria: create, update, and delete operations preserve stable
    ids and monotonically increasing event order in tests.

### 1.2. Actor context, authorisation, and visibility

- [ ] 1.2.1. Introduce request-scoped actor resolution.
  - Support at least anonymous, user, app, and installation actors.
  - Map inbound tokens or headers to a deterministic actor fixture for tests.
- [ ] 1.2.2. Model repository membership and permissions.
  - Add repository roles, team membership, and installation permissions with
    enough fidelity to gate common collaboration operations.
  - Completion criteria: permission tests distinguish read, write, triage,
    maintain, and admin behaviour where relevant.
- [ ] 1.2.3. Expose actor context to REST and GraphQL.
  - Replace the current "first user in the store" behaviour with actor-aware
    selectors for `viewer`, `/user`, memberships, and repository access.

### 1.3. Shared mutation, event, and fixture framework

- [ ] 1.3.1. Add domain actions and reducers for write workflows.
  - Create typed actions for create, update, close, reopen, merge, review, and
    comment operations instead of mutating route-local response objects.
- [ ] 1.3.2. Add event emission and timeline recording.
  - Record state changes as timeline events that can back REST timelines,
    GraphQL timeline items, notifications, and webhooks.
- [ ] 1.3.3. Publish fixture builders for collaboration graphs.
  - Provide helpers for creating repositories with branches, issues, pull
    requests, reviews, and checks in one place.
  - Completion criteria: new slices can add scenario tests without hand-writing
    unrelated entities.

## 2. Repository and issue collaboration slice

This phase delivers the first complete user-visible slice: a repository with
issues, comments, labels, assignees, and timelines that can be created and
manipulated through both REST and GraphQL.

### 2.1. Repository graph and content operations

- [ ] 2.1.1. Complete repository, ref, and content scriptability. Requires
      1.1.1 and 1.3.1.
  - Support repository lookup, branch lookup, commit lookup, file contents, and
    tree traversal from shared commit and blob entities.
  - Fix existing broken or partial content behaviours such as tree lookup and
    branch scoping.
- [ ] 2.1.2. Add writeable branch and file mutation helpers. Requires 2.1.1.
  - Support creating refs, updating refs, and committing file content changes
    within simulator constraints.
  - Completion criteria: tests can create a feature branch, modify files, and
    observe those changes through REST and GraphQL reads.

### 2.2. Issues, comments, labels, and assignees

- [ ] 2.2.1. Implement issue lifecycle APIs. Requires 1.1.2, 1.2.2, and 1.3.1.
  - Cover create, list, fetch, update, close, and reopen for issues.
  - Keep REST and GraphQL views consistent for title, body, state, labels,
    assignees, and author.
- [ ] 2.2.2. Implement issue comments and reactions. Requires 2.2.1.
  - Support create, edit, delete, and list operations for comments.
  - Add timeline items so comment history is visible through both surfaces.
- [ ] 2.2.3. Implement labels, milestones, and assignee management. Requires
      2.2.1.
  - Completion criteria: a repository fixture can create labelled issues,
    reassign them, and query the same state through REST and GraphQL.

### 2.3. Issue timeline and notification parity

- [ ] 2.3.1. Add issue timeline REST and GraphQL views. Requires 1.3.2 and
      2.2.2.
  - Expose a stable subset of timeline items: opened, closed, reopened,
    labelled, unlabelled, assigned, unassigned, commented.
- [ ] 2.3.2. Add issue subscription and notification state. Requires 2.3.1.
  - Model thread subscriptions, unread state, and per-actor notification
    delivery for issues.

## 3. Pull request lifecycle slice

This phase delivers a complete branch-to-pull-request flow. The outcome is that
tests can create a feature branch, open a pull request, query it through REST
and GraphQL, and observe changed files and discussion metadata.

### 3.1. Pull request domain model and read surfaces

- [ ] 3.1.1. Add first-class pull request entities. Requires 1.1.2 and 2.1.2.
  - Model base ref, head ref, author, draft state, merge state, requested
    reviewers, linked issue number, and timeline events.
- [ ] 3.1.2. Implement pull request list and detail APIs. Requires 3.1.1.
  - Cover the common REST list/detail endpoints and GraphQL fields used by
    repository and pull request views.
- [ ] 3.1.3. Implement diff, file, and commit views. Requires 2.1.1 and 3.1.1.
  - Expose changed files, commit list, compare data, and head/base metadata.
  - Completion criteria: a test can open a pull request from a feature branch
    and inspect its file delta through both APIs.

### 3.2. Pull request mutations and state transitions

- [ ] 3.2.1. Implement create, update, close, and reopen pull request
      operations. Requires 3.1.2.
- [ ] 3.2.2. Implement reviewer request management. Requires 1.2.2 and 3.2.1.
  - Support requesting and removing individual reviewers and teams.
- [ ] 3.2.3. Add pull request timeline items. Requires 1.3.2 and 3.2.1.
  - Include opened, converted to draft, ready for review, closed, reopened,
    reviewer requested, and review submitted events.

## 4. Reviews and review threads slice

This phase turns pull requests into genuinely reviewable artefacts. The goal is
full scriptability for review requests, submitted reviews, inline comments, and
thread resolution.

### 4.1. Review submission and decision states

- [ ] 4.1.1. Implement pull request review entities and submission flow.
      Requires 3.2.1.
  - Support pending reviews, submitted reviews, dismissal, and the common
    review conclusions: comment, approve, and request changes.
- [ ] 4.1.2. Expose review summaries through REST and GraphQL. Requires 4.1.1.
  - GraphQL coverage should include the fields needed by review-centric user
    interfaces, not only list totals.
- [ ] 4.1.3. Add requested-reviewer and latest-review state selectors. Requires
      4.1.1.
  - Completion criteria: tests can assert review state transitions without
    reconstructing them from raw comments.

### 4.2. Inline comments and review threads

- [ ] 4.2.1. Implement pull request review comments on diffs. Requires 3.1.3
      and 4.1.1.
  - Support side, line, commit, and path metadata for inline review comments.
- [ ] 4.2.2. Implement review thread grouping and resolution. Requires 4.2.1.
  - Provide thread identity, resolved state, reply chains, and actor
    attribution.
- [ ] 4.2.3. Expose review threads through GraphQL and REST. Requires 4.2.2.
  - Completion criteria: review UI tests can render threads, replies, and
    resolution state from simulator data alone.

### 4.3. Team review and membership-driven behaviour

- [ ] 4.3.1. Implement team review request resolution. Requires 1.2.2 and
      4.1.1.
  - Allow a team review request to expand into eligible reviewers according to
    fixture membership.
- [ ] 4.3.2. Add organisation team and membership GraphQL resolvers. Requires
      4.3.1.
  - Replace the current empty `teams` and `membersWithRole` placeholders with
    store-backed data.

## 5. Checks, mergeability, and merge outcomes slice

This phase makes pull requests operationally useful. A pull request should be
able to collect status, become mergeable or blocked, and change repository
state when merged.

### 5.1. Commit status and check run scriptability

- [ ] 5.1.1. Implement commit status history. Requires 2.1.1 and 3.1.3.
  - Replace fixed combined-status responses with mutable per-sha status state.
- [ ] 5.1.2. Implement check suites and check runs. Requires 5.1.1.
  - Cover the read and write operations needed by GitHub Apps and deployment
    tooling.
- [ ] 5.1.3. Add GraphQL status and check summaries. Requires 5.1.2.
  - Completion criteria: a pull request can show failing checks and later show
    success after scripted updates.

### 5.2. Mergeability, branch protection, and merge execution

- [ ] 5.2.1. Implement mergeability selectors. Requires 4.2.3 and 5.1.2.
  - Consider open review requests, requested changes, failing checks, merge
    conflicts, and draft state.
- [ ] 5.2.2. Implement a minimal branch protection model. Requires 1.2.2 and
      5.2.1.
  - Support required reviews, required checks, and force-push constraints.
- [ ] 5.2.3. Implement merge operations. Requires 5.2.2.
  - Support merge, squash, and rebase strategies with observable post-merge
    effects on refs, timelines, and pull request state.
  - Completion criteria: tests can merge an approved pull request and observe
    repository, issue, and webhook changes.

## 6. App-facing integration slice

This phase makes the simulator useful for realistic integration tests for
GitHub Apps, cataloguers, and platform services that consume GitHub events and
query mixed REST and GraphQL surfaces.

### 6.1. Webhooks and delivery history

- [ ] 6.1.1. Emit webhook events for issue and pull request activity. Requires
      1.3.2, 2.2.1, and 3.2.1.
  - Cover the common events needed by automation: `issues`,
    `issue_comment`, `pull_request`, `pull_request_review`, and
    `pull_request_review_comment`.
- [ ] 6.1.2. Add webhook delivery inspection and replay helpers. Requires
      6.1.1.
  - Provide a deterministic delivery log so tests can assert payload order and
    contents.

### 6.2. REST and GraphQL parity sweep for common workflows

- [ ] 6.2.1. Add a capability matrix for the supported collaboration slices.
      Requires 2.3.2, 4.3.2, and 5.2.3.
  - Document which REST endpoints and GraphQL fields are fully scriptable,
    placeholder-only, or intentionally unsupported.
- [ ] 6.2.2. Add end-to-end scenario suites. Requires 6.1.2.
  - Cover at least:
    - issue creation and triage
    - pull request open, review, and merge
    - review thread resolution
    - failing check to passing check transition
    - webhook delivery for each scenario
- [ ] 6.2.3. Publish fixture recipes for common consumers. Requires 6.2.2.
  - Provide ready-made builders for GitHub App, Backstage catalog, and review
    workflow scenarios.

## Sequencing notes

- Phase 1 is intentionally narrow. Any task that does not directly unlock
  writeable collaboration slices should be deferred.
- Phase 2 is the first complete collaboration slice and should land before pull
  request work begins.
- Phase 3 depends on repository and branch mutation support from Phase 2.
- Phase 4 depends on the pull request domain model from Phase 3, but its team
  membership work also strengthens the actor model from Phase 1.
- Phase 5 should not begin until pull request reviews are scriptable, because
  mergeability without review state produces misleading results.
- Phase 6 should only sweep parity for the slices already delivered. It should
  not become a back door for adding unrelated feature areas.

## Definition of done for the roadmap

The roadmap should be treated as complete when all of the following are true:

- Repository, issue, pull request, review, and merge workflows are writeable
  and queryable through both REST and GraphQL.
- Actor-aware permissions replace the current fixed-user shortcuts.
- Webhooks and notifications are driven by the same domain events as REST and
  GraphQL reads.
- The package documentation explains supported slices and their limits clearly.
- End-to-end tests prove the vertical slices work without hand-editing internal
  store state between steps.
