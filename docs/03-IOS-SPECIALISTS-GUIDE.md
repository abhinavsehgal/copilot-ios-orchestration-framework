# 03 — iOS Specialists Guide

The eight default iOS specialist agents shipped in this framework, what each one owns, when to add custom ones, and the universal rules every specialist must follow.

## The default roster

```
.github/agents/
├── <your-app>-orchestrator.md      ← coordinator (no implementation)
├── ios-ui.md                        ← SwiftUI / UIKit / Auto Layout / accessibility
├── ios-data.md                      ← CoreData / SwiftData / Realm / file storage
├── ios-network.md                   ← URLSession / async-await / Combine / cert pinning
├── ios-tests.md                     ← XCTest / XCUITest / snapshot tests
├── ios-release.md                   ← Fastlane / TestFlight / App Store Connect
├── ios-privacy.md                   ← REVIEW-ONLY: Info.plist / ATT / nutrition labels
├── ios-perf.md                      ← Instruments / hangs / memory / launch time
└── ios-bg.md                        ← BGTaskScheduler / silent push / lifecycle
```

Eight specialists is a lot for a small app. **Drop the ones you don't need.** A simple app might keep just `ios-ui`, `ios-data`, `ios-network`, `ios-tests`. A pre-launch app skips `ios-release`. A no-background app skips `ios-bg`.

## Per-specialist scope tables

### `<your-app>-orchestrator`

The coordinator. Has NO Edit/Write tools — it can only Read, Grep, Glob, Bash. Its job is to receive the user's task, decompose it into specialist handoffs, validate each return, and aggregate results.

| Owns | Doesn't own |
|---|---|
| Task decomposition | Implementation (delegates) |
| Specialist selection | Editing files |
| Handoff schema validation | Running tests directly (delegates to `ios-tests`) |
| Return-block validation (no evidence → reject) | Building the app (delegates to `ios-release` or `/verify-build`) |

**Frontmatter:**
```yaml
tools: Read, Grep, Glob, Bash, mcp__github__*
target: vscode, github-copilot
disable-model-invocation: true        # only invoked explicitly
user-invocable: true
```

### `ios-ui`

SwiftUI views, UIKit view controllers, Auto Layout, accessibility, dark mode, dynamic type, localization rendering, custom drawing.

| Owns | Doesn't own |
|---|---|
| View hierarchy + layout | Data fetching (handoff to `ios-data`) |
| `@State` / `@StateObject` / `@ObservedObject` patterns | Network calls (handoff to `ios-network`) |
| Accessibility labels, traits, dynamic type | Background lifecycle (handoff to `ios-bg`) |
| Storyboards / XIBs (legacy UIKit screens) | Performance profiling (handoff to `ios-perf`) |

**Auto-loads:** `.github/instructions/swiftui.instructions.md` for any `Sources/**/Views/**/*.swift`, `*.xib`, `*.storyboard`.

### `ios-data`

CoreData stack management (containers, contexts, merge policies), SwiftData migrations, Realm models, FileManager / Documents / Caches, Keychain (data-only — auth flows go to `ios-network`).

| Owns | Doesn't own |
|---|---|
| `NSManagedObjectContext` pattern (main vs background vs perform) | Network sync (handoff to `ios-network`) |
| CoreData / SwiftData migration scripts | View bindings (handoff to `ios-ui`) |
| File / disk storage strategies | Cryptographic key generation (handoff to `ios-network` for TLS, `ios-privacy` for entitlement-protected data) |
| Database schema versioning | Performance benchmarks (handoff to `ios-perf`) |

### `ios-network`

URLSession, async-await networking, Combine publishers for streaming, Alamofire (if used), cert pinning, GraphQL clients, websockets, background fetch tasks (the network side; lifecycle goes to `ios-bg`).

| Owns | Doesn't own |
|---|---|
| Request/response serialization | Storing fetched data (handoff to `ios-data`) |
| Cert pinning + TLS configuration | Display of network errors (handoff to `ios-ui`) |
| Auth refresh flows + Keychain credentials | Privacy declarations (handoff to `ios-privacy`) |
| Network error retry/backoff strategies | App lifecycle wake-from-background (handoff to `ios-bg`) |

### `ios-tests`

XCTest unit tests, XCUITest UI tests, snapshot tests (e.g. SnapshotTesting library), test plans, testability annotations.

| Owns | Doesn't own |
|---|---|
| Writing + maintaining tests | Implementation under test (delegates back to ui/data/network) |
| Test plans (`.xctestplan`) | Production code coverage (informs all specialists) |
| Snapshot generation + replay | App Store crash report triage (handoff to `ios-perf`) |
| Mocking frameworks + protocol-oriented test doubles | Release notes from test results (handoff to `ios-release`) |

### `ios-release`

Fastlane lanes, TestFlight builds, App Store Connect API, code signing automation, version + build number bumping, archive + IPA upload, App Store screenshots/metadata.

| Owns | Doesn't own |
|---|---|
| Fastfile + lane logic | Crash post-release triage (handoff to `ios-perf`) |
| Build number bumping policy | Privacy nutrition labels (handoff to `ios-privacy`) |
| TestFlight + App Store submission | Release notes copy (collaborates with product/marketing) |
| Provisioning profile rotation | Code signing certificate generation (locked: per-team manual process) |

### `ios-privacy` — REVIEW-ONLY

`Info.plist` usage descriptions, App Tracking Transparency, privacy nutrition labels, App Store Required Reasons API declarations, COPPA / GDPR / DPDP compliance, third-party SDK privacy manifests.

**REVIEW-ONLY = `tools: Read, Grep, Glob` (no Edit/Write).** This is a runtime lock — Copilot's harness physically blocks file edits from this agent. The agent reads, finds risks, files findings; humans (or implementation specialists) make the changes.

| Owns | Doesn't own |
|---|---|
| `Info.plist` usage description audit | Editing `Info.plist` (other specialists do that, then `ios-privacy` re-reviews) |
| ATT prompt timing audit | Implementation of permission flows |
| Privacy nutrition label review | Filing in App Store Connect (handoff to `ios-release`) |
| Third-party SDK privacy manifest verification | Removing SDKs (handoff to `ios-release` + `ios-data` etc.) |
| COPPA / GDPR / DPDP risk surfacing (never legal advice) | Negotiating with App Store reviewer (humans only) |

### `ios-perf`

Instruments profiles (Time, Allocations, Leaks, Hangs), memory + retain cycles, launch time, MetricKit reports, Xcode Organizer crashes, energy log.

| Owns | Doesn't own |
|---|---|
| Instruments-driven optimization | Writing the optimized code (handoff to `ios-ui` / `ios-data` / `ios-network`) |
| MetricKit + Organizer crash triage | Bug reports → tickets (handoff to standard PR flow) |
| Launch time + cold-start budget | Release timing (handoff to `ios-release`) |
| Memory leak hunts (Leaks instrument, retain cycle audit) | Test fixture for the leak (handoff to `ios-tests`) |

### `ios-bg`

Background tasks (BGTaskScheduler), silent push notifications, app lifecycle (UIScene / UIApplication delegate), state restoration, background URL sessions (the lifecycle side; the URL session itself is `ios-network`).

| Owns | Doesn't own |
|---|---|
| BGTaskScheduler registration + earliest-begin-date | Network request inside the background task (handoff to `ios-network`) |
| Silent push handling | Push registration / APNs token (handoff to `ios-network`) |
| Scene + app lifecycle transitions | UI shown when returning from background (handoff to `ios-ui`) |
| State restoration (when supported) | Data persistence during background (handoff to `ios-data`) |

## When to add a custom specialist

Add a new specialist when:
- A coherent domain has > 3 recurring tasks per quarter
- Cross-cutting handoffs would otherwise pollute existing specialists' scope
- The domain has its own gotchas worth path-globbed instructions

Common iOS additions:

| Specialist | When to add |
|---|---|
| `ios-watch` | watchOS companion app or complications, > 1 person-week of watch work |
| `ios-vision` | visionOS companion or full visionOS app |
| `ios-tv` | tvOS companion |
| `ios-iap` | StoreKit 2 / subscriptions / receipt validation are core to the app |
| `ios-deep-link` | Universal Links / URL schemes / SiriKit intents are heavy |
| `ios-widgets` | WidgetKit and Live Activities are part of the product |
| `ios-arkit` | AR features beyond a one-time camera prompt |
| `ios-camera` | Custom camera UI, AVCaptureSession heavy |
| `ios-mlkit` | CoreML / Vision / Speech / NaturalLanguage frameworks active |
| `ios-l10n` | Localization is heavy, RTL critical, or multi-region App Store |

## Universal rules every specialist must follow

These appear verbatim in the body of every specialist agent template:

1. **Validate the incoming handoff before acting.** Refuse vague delegations (no `failure_condition`, no `acceptance_criteria`, no `evidence`).
2. **Honor the universal evidence rule.** Every claim cites a path:line, Info.plist key, schema, or test output. Mark assumptions as `confidence: low`.
3. **Read the path-globbed instruction file before editing.** When you're about to edit `Sources/Views/Profile.swift`, the auto-load surfaces `swiftui.instructions.md`. Read its Hard Rules before touching the file.
4. **Return a structured `return:` block** with `verified_claims`, `unverified_claims`, `files_changed`, `tests_run`, `do_not_pass_downstream_without_verification`, and `status` (one of `completed | partial | claim_rejected | needs_clarification`).
5. **Definition of Done** — explicitly list what's done before stopping. If `tests_run` is empty for a code change, the orchestrator will reject the return.

## Cross-links

- [`04-HANDOFF-SCHEMA.md`](04-HANDOFF-SCHEMA.md) — the contract every specialist enters and exits
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — what the auto-loaded instruction files look like
- [`08-IOS-COMMON-PITFALLS.md`](08-IOS-COMMON-PITFALLS.md) — gotchas each specialist needs to know about
