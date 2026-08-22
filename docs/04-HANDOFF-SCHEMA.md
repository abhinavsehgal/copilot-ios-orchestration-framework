# 04 — Handoff Schema

The bidirectional contract every orchestrator-to-specialist delegation must use, with iOS examples.

## Why a structured schema

Without it, delegations are free-form prompts. The orchestrator says "fix the avatar bug" and the specialist makes assumptions. Specialist returns "fixed it" and the orchestrator can't tell what was actually verified. Hallucinations propagate hop-to-hop with no friction to catch them.

The schema introduces friction at every hop:
- The orchestrator can't issue a vague delegation (the schema requires `goal` + `scope` + `evidence` + `failure_condition` + `acceptance_criteria`).
- The specialist can't return a vague result (the schema requires `verified_claims` + `unverified_claims` + `files_changed` + `tests_run` + `status`).
- The orchestrator validates every return before marking the parent task done.

## The outbound (orchestrator → specialist) shape

```yaml
handoff:
  version: 1
  from: <your-app>-orchestrator
  to: ios-data
  hop: 1
  goal: |
    Add a `streak` Int16 attribute to the User entity in Profile.xcdatamodeld
    so /track-streak feature can persist daily login counts.

  scope:
    paths_in:
      - Sources/Profile/Profile.xcdatamodeld
      - Sources/Profile/UserModel+Streak.swift
    paths_out:
      - Sources/Views/**
      - Sources/Networking/**

  constraints:
    - iOS 15 minimum — no SwiftData (iOS 17+)
    - Migrate from current schema version 4 → 5 with lightweight migration
    - DO NOT change the `User.email` attribute (currently primary identifier)

  evidence:
    - Profile.xcdatamodeld/Profile.xcdatamodel/contents:User (current schema)
    - docs/ai-context/coredata-schema.md:42 (versioning policy)
    - .github/instructions/coredata.instructions.md (rules for context patterns)

  failure_condition: |
    If the existing schema already has a `streak` attribute on any entity
    (User or otherwise), the orchestrator's hypothesis is wrong — stop and
    return claim_rejected with the file:line citation.

  acceptance_criteria:
    - New `streak` Int16 attribute on User entity, default 0, non-optional
    - New schema version (5) with mapping model from version 4
    - `xcodebuild test -only-testing:UserModelTests` passes
    - Migration test added: load v4 store, open with v5 model, verify streak == 0

  budget:
    max_files_changed: 6
    max_hops: 2
    max_lines_added: 200
```

## The inbound (specialist → orchestrator) shape

```yaml
return:
  version: 1
  from: ios-data
  to: <your-app>-orchestrator
  hop: 1
  status: completed   # completed | partial | claim_rejected | needs_clarification

  verified_claims:
    - "Added streak Int16 attribute on User entity (Profile.xcdatamodeld/Profile.xcdatamodel/contents:User:streak)."
    - "Created schema version 5 with mapping model from v4."
    - "All UserModelTests pass via xcodebuild test -only-testing:UserModelTests (full output in tests_run.results)."
    - "Lightweight migration verified: existing user records have streak = 0 post-migration (UserMigrationTests.testV4ToV5MigrationStreakDefault)."

  unverified_claims: []

  do_not_pass_downstream_without_verification: []

  files_changed:
    - Sources/Profile/Profile.xcdatamodeld/Profile 5.xcdatamodel/contents
    - Sources/Profile/UserModel+Streak.swift  # new file
    - Sources/Profile/UserModel+Streak.swift  # extension on existing User class for streak helpers
    - Tests/Profile/UserMigrationTests.swift  # new test file

  tests_run:
    - command: xcodebuild test -only-testing:UserModelTests -workspace MyApp.xcworkspace -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'
      result: pass
      tail: "Test Suite 'UserModelTests' passed at 2026-05-07 14:32:11 — 8 tests, 0 failures"
    - command: xcodebuild test -only-testing:UserMigrationTests
      result: pass
      tail: "testV4ToV5MigrationStreakDefault passed (0.124s)"

  budget_consumed:
    files_changed: 4
    hops: 1
    lines_added: 87

  follow_ups:
    - "Schema version 5 is now the latest. Bump CoreData store version constant in AppDelegate.swift if you have one (didn't find one in scope)."
    - "Add `streak` to the iCloud sync layer if applicable (not in scope; flag for ios-network)."
```

## Status values

| Status | When to use | What the orchestrator does |
|---|---|---|
| `completed` | All `acceptance_criteria` met, all `verified_claims` populated, all tests in `tests_run` passed | Mark parent task done; pass `follow_ups` to user |
| `partial` | Some criteria met but not all (e.g. `tests_run` empty for a code change) | Reject the return; ask the specialist to complete it |
| `claim_rejected` | The orchestrator's premise turned out wrong (e.g. `failure_condition` was observed) | Stop the parent task; surface to user with the triggering evidence |
| `needs_clarification` | The handoff was ambiguous in some specific way the specialist can't resolve from project state | Orchestrator asks user the specific clarifying question; re-issue handoff |

## The "no evidence, no claim" rule, applied

Bad return:
```yaml
verified_claims:
  - "Streak attribute added"
  - "Migration works"
  - "Tests pass"
```

Good return:
```yaml
verified_claims:
  - "Added streak Int16 attribute on User entity (Profile.xcdatamodeld/Profile.xcdatamodel/contents:User:streak)."
  - "Migration v4 → v5 verified by UserMigrationTests.testV4ToV5MigrationStreakDefault (passes; tests_run[1])."
  - "All 8 UserModelTests pass (tests_run[0])."
```

Each claim cites a specific path, schema location, or test name. The orchestrator can verify each citation. Hallucinations have nowhere to hide.

## When to use multi-hop

Multi-hop = the specialist hands off to another specialist mid-task. iOS example:

> User: "Show user streak in the profile screen and persist it."

Orchestrator → `ios-data` (add streak attribute, persist). 
`ios-data` returns `completed` with follow-up: *"Schema is updated; UI binding needs ios-ui."*
Orchestrator → `ios-ui` (display streak in ProfileView).
`ios-ui` reads the schema location from the previous return's `verified_claims`, builds the SwiftUI binding, returns `completed`.
Orchestrator aggregates both and reports done.

**Hop limit:** soft cap of 5 hops. If you hit hop 5, escalate to user — the task is bigger than the orchestrator initially modeled.

## Optional fields added in v1.1.0 (additive — `schema_version` stays 1)

**Outbound**, for multi-repo workspaces ([`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md)) — omit in single-repo projects:

```yaml
  repo: <repo name from workspace.json — the ONLY repo this handoff may edit, e.g. ios-app>
  contract_impact:
    level: <none | additive | breaking>
    contracts: [<names from CONTRACTS.md — an API spec, or a shared Swift package>]
    consumers_to_update: [<repo names — required when level != none>]
```

**Outbound**, any project — the evidence-confidence class from [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md) may be used in place of the
three-value `confidence:` on a claim (`verified-code` / `verified-schema` / `verified-test` /
`verified-git` / `documented-unverified` / `historical` / `unknown`). Specialists treat anything
that is not `verified-*` exactly as they treat `confidence: low`. An `Info.plist` key you read is
`verified-schema`; a key an instruction file *says* is there is `documented-unverified`.

**Inbound**, for multi-repo workspaces:

```yaml
  contracts_changed:
    - contract: <name>
      change: <one line>
      backward_compatible: <true | false>
      consumers_grepped: [<repo>:<path>, …]   # e.g. ios-app:Sources/Networking/OrderResponse.swift
```

**Inbound**, any project — `deferred_work:` lists anything the specialist is *not* doing that
someone must (each item with the backlog file it was appended to). A return that names deferred work
without a backlog path is incomplete (Chapter 12).

## Versioning

- `schema_version: 1` is current.
- Additive changes (new optional fields) keep version 1.
- Breaking changes (renaming fields, changing field semantics) bump the version.
- Bumps require updating HANDOFF_SCHEMA.md, the orchestrator, and every specialist file in the same PR.

## Cross-links

- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — who each specialist is, when to delegate to them
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — what specialists auto-load when they reach the file paths in `paths_in`
- The `templates/HANDOFF_SCHEMA.md.template` — drop into your project at `docs/ai-context/HANDOFF_SCHEMA.md`
