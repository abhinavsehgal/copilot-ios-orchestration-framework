# 08 — iOS Common Pitfalls

Twenty-plus hard-won iOS-specific lessons. Read this before bootstrapping.

## Framework / configuration pitfalls (1-7)

### Pitfall 1: Putting the orchestrator's persona in `.github/copilot-instructions.md`

The repository-wide instructions file is auto-loaded into EVERY Copilot interaction — including inline completions while you're typing in Xcode. Loading the orchestrator's full persona there:
- Slows down inline completions
- Pollutes code-review prompts with delegation instructions
- Forces every interaction through the orchestrator's "delegate everything" body

**Right answer:** Keep `.github/copilot-instructions.md` thin (golden rules + workflow + cross-links). The orchestrator agent is invoked explicitly via `@<your-app>-orchestrator`.

### Pitfall 2: REVIEW-ONLY agents with auto-Edit privileges

If you set `tools: Read, Grep, Glob, Edit, Write` on `ios-privacy`, your privacy reviewer can now silently change `Info.plist`. The whole point of REVIEW-ONLY is the runtime lock — Copilot's harness physically blocks Edit if Edit isn't in `tools:`.

**Right answer:** REVIEW-ONLY agents have only `Read, Grep, Glob` (and optionally `Bash` for read-only commands). Never include `Edit`, `Write`, or `MultiEdit`.

### Pitfall 3: Treating documentation enforcement as runtime enforcement

The handoff schema, the universal evidence rule, the failure_condition observation — these are **documentation discipline**. They work because agents follow their instructions. They do NOT physically prevent a misbehaving agent from emitting a malformed handoff.

What IS runtime-enforced (by Copilot's harness):
- `tools:` allowlists
- `disable-model-invocation:`
- `user-invocable:`
- `target:` (vscode / github-copilot)
- `mcp-servers:` per-agent MCP scoping
- `applyTo:` glob auto-loading
- Body length cap (~30,000 chars)

**Right answer:** be honest about which layer enforces what. If a documentation rule keeps being broken, escalate it to the runtime layer where possible.

### Pitfall 4: `applyTo:` globs that don't match your actual file paths

Spending an hour writing `swiftui.instructions.md` only to realize your views live at `Targets/MyApp/Sources/Views/`, not `Sources/Views/`. The instruction file never auto-loads.

**Right answer:** before writing instructions, check your actual file structure. Run `find . -name '*.swift' -path '*Views*' | head -10` to see what real paths look like. Use those in `applyTo:`. Test by editing one of those files and asking Copilot Chat *"what instruction files apply to the active file?"*

### Pitfall 5: Not testing instruction files in code review

Code review reads only the first ~4,000 chars of each instruction file. If your most-violated rule is on line 200, code review never sees it.

**Right answer:** when authoring an instruction file, ask yourself *"if a PR violates this rule, will code review catch it?"* If yes, the rule must be in the first 4k. Run `wc -c < .github/instructions/foo.instructions.md` to check.

### Pitfall 6: Letting `docs/` rot without a librarian

After a year, `docs/` has 50+ files: sprint reports, post-mortems, outdated architecture decisions, dated migration plans. Engineers can't tell which are authoritative. Worse, Copilot reads them all and produces drift.

**Right answer:** add a `context-librarian` specialist whose job is exactly this. Schedule a quarterly cleanup pass. Use the three-tier system from [`07-FOLDER-STRUCTURE.md`](07-FOLDER-STRUCTURE.md). Move stale to `docs/_archive/<YYYY-MM>/`. Never delete.

### Pitfall 7: Bootstrap on a repo with existing `.github/copilot-instructions.md`

If your team has been adding to `.github/copilot-instructions.md` for months, BOOTSTRAP-PROMPT must NOT silently overwrite that work. The framework's bootstrap includes pre-flight safety checks: snapshot to `.github-pre-bootstrap-backup/`, naming-collision detection, `applyTo:` glob conflict detection, drift detection on existing instructions, and a decision gate that STOPS for explicit user confirmation per file.

**Right answer:** even on apparent greenfield projects, run the pre-flight. Cost is 30 seconds; benefit is never silently destroying team work.

## iOS-specific pitfalls (8-22)

### Pitfall 8: Not pinning iOS deployment target in instruction files

A SwiftUI rule that's correct on iOS 17+ might be wrong on iOS 15 (or vice versa). Without pinning, Copilot generates code using whatever it knows latest.

**Right answer:** every `swiftui.instructions.md` / `concurrency.instructions.md` / etc. starts with the deployment target: *"This project supports iOS 15.0 minimum. Do not use SwiftData (iOS 17+), `@Observable` macro (iOS 17+), or `@Bindable` (iOS 17+) without `@available` guards."*

### Pitfall 9: Background `viewContext.save()` (CoreData)

Calling `viewContext.save()` from a background thread crashes with `unrecognized selector` or worse, silently corrupts the persistent store. The viewContext is `@MainActor`-bound.

**Right answer:** instruction file `coredata.instructions.md` Hard rule: *"`viewContext` must only be used on the main thread. Use `container.performBackgroundTask { ctx in ... }` for non-UI work, or `ctx.perform { }` to switch to the context's queue. Set `automaticallyMergesChangesFromParent = true` on `viewContext` for cross-context propagation."*

### Pitfall 10: Strong `self` capture in `Task { }` inside `@MainActor` views

`Task { await self.fetchData() }` in a SwiftUI view's `.onAppear` creates a retain cycle with the view's `@StateObject`. The view never deallocates; memory grows.

**Right answer:** instruction file `concurrency.instructions.md` Hard rule: *"In view-scoped Task closures, either use `[weak self]`, extract to a free function with explicit parameters, or use `.task { }` view modifier (auto-cancels on view-disappear). Never write `Task { await self.foo() }` in `.onAppear`."*

### Pitfall 11: Missing `Info.plist` usage descriptions

Adding camera / photo library / location / contacts access without the corresponding `NSXxxUsageDescription` key crashes the app at runtime AND fails App Store Review with rejection notice.

**Right answer:** instruction file `info-plist.instructions.md` covers every usage description key. The `ios-privacy` REVIEW-ONLY agent audits Info.plist on every PR that touches an entitlement or capability.

| New API used | Required Info.plist key |
|---|---|
| Camera (AVCapture) | `NSCameraUsageDescription` |
| Photo library | `NSPhotoLibraryUsageDescription` (read), `NSPhotoLibraryAddUsageDescription` (write) |
| Location | `NSLocationWhenInUseUsageDescription` and/or `NSLocationAlwaysAndWhenInUseUsageDescription` |
| Contacts | `NSContactsUsageDescription` |
| Microphone | `NSMicrophoneUsageDescription` |
| FaceID/TouchID | `NSFaceIDUsageDescription` |
| Tracking (IDFA) | `NSUserTrackingUsageDescription` + ATT prompt |
| Calendar | `NSCalendarsUsageDescription` (read), `NSCalendarsFullAccessUsageDescription` (iOS 17+) |
| Health | `NSHealthShareUsageDescription`, `NSHealthUpdateUsageDescription` |
| Bluetooth | `NSBluetoothAlwaysUsageDescription` |

### Pitfall 12: ATT prompt before any tracking framework loads

App Store rejects apps that initialize an ad SDK / analytics SDK before requesting tracking authorization. The ATT prompt must come first; SDK init only after `.authorized`.

**Right answer:** instruction file `info-plist.instructions.md` Hard rule: *"For any framework using IDFA: (1) Add `NSUserTrackingUsageDescription` to Info.plist, (2) Call `ATTrackingManager.requestTrackingAuthorization` BEFORE the framework initializes, (3) Init the framework only on `.authorized` state, (4) Document this sequence in PR description."* `ios-privacy` audits this on PRs.

### Pitfall 13: Code signing files committed to git

Committing `.cer`, `.p12`, `.mobileprovision`, `.cer`, or `Match` keychain files leaks signing identity. Even after deletion, the files are in git history forever — must rotate every signing certificate.

**Right answer:** `.gitignore` includes `*.cer`, `*.p12`, `*.mobileprovision`, `Profiles/*`, `keychain-*`, `*.keychain-db`. The `/commit-push-pr` slash prompt explicitly refuses to stage any path matching those patterns. If discovered after the fact: rotate certs (treat as compromised), use git-filter-repo or BFG to scrub history, force-push, coordinate with team.

### Pitfall 14: Shipping `print()` and `os_log` debug logs to App Store

Verbose logging in production:
- Spams Console.app for users
- Wastes energy
- Can leak PII into device logs (visible to anyone with the device)
- App Review may reject for excessive logging

**Right answer:** instruction file (e.g. `logging.instructions.md`) Hard rule: *"Production builds use `Logger` from `OSLog` with `privacy: .private` markers for any user data. Debug-only logs are wrapped in `#if DEBUG`. No `print()` in shipping code."*

### Pitfall 15: Force-unwrapping in public API surface

`func getUser() -> User { users.first! }` crashes when the array is empty. Force unwrapping in public API turns recoverable nil cases into crashes — hits Crashlytics top issues consistently.

**Right answer:** instruction file Hard rule: *"No `!` (force unwrap) in any public API of any module/framework. Internal/private code may use `!` only when nil is provably impossible (and that proof is commented inline). PR review blocks `!` in public surface."*

### Pitfall 16: Missing `@MainActor` on UI mutation methods

Mutating `@Published` properties or UIView state from a background thread can cause silent UI corruption (the change appears with random delay) or hard crashes (if UIKit notices).

**Right answer:** instruction file `concurrency.instructions.md` Hard rule: *"Methods that mutate `@Published`, `@State`, or any UIView/UIViewController state are annotated `@MainActor`. Async functions returning to UI use `await MainActor.run { }` or `Task { @MainActor in }`."*

### Pitfall 17: New SDK without privacy manifest (iOS 17+)

Adding a third-party SDK that uses any "Required Reasons API" (file timestamp, user defaults read, system boot time, disk space, active keyboards) without a corresponding `PrivacyInfo.xcprivacy` file in the SDK → App Store rejection on submission.

**Right answer:** `ios-privacy` REVIEW-ONLY agent runs a check on every PR that adds a Pod / SPM dependency: *"Does this SDK ship a `PrivacyInfo.xcprivacy` manifest declaring its Required Reasons API usage?"* If no, surface the risk before merge.

### Pitfall 18: Universal Links domain not in apple-app-site-association

Implementing Universal Link handling in `Application(_:continue:restorationHandler:)` without registering the domain in your apple-app-site-association file → links open in Safari instead of your app.

**Right answer:** instruction file (e.g. `deeplinks.instructions.md` if you have one) Hard rule: *"Adding a Universal Link path requires (1) entitlement update, (2) AASA file update on web side, (3) AASA-validator check, (4) on-device test. The PR description must list all four."*

### Pitfall 19: Copilot has no programmable lifecycle hooks

If you've come from Claude Code or other harnesses expecting `PreToolUse` / `PostToolUse` / `Stop` hooks, you won't find them on Copilot. Copilot has:

- **Declarative auto-loading** via `applyTo:` (the platform-native equivalent of PreToolUse rule-surfacing)
- **Manually-invoked prompts** via slash commands
- **Definition-of-Done discipline** in agent instructions
- **IDE-level auto-fix** and pre-commit hooks (out of framework scope)

See [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md). Don't try to build Copilot hook scripts — there's no event to wire to.

### Pitfall 20: Cloud Agent without code signing access

The Cloud Agent runs in GitHub Actions. To run `xcodebuild archive` it needs signing certificates and provisioning profiles. By default it has neither.

**Right answer:** if you want Cloud Agent to do `archive` work, set up signing via [Match](https://docs.fastlane.tools/actions/match/) with secrets in GitHub Actions (or App Store Connect API key). Document in `docs/ai-context/release-process.md`. If you don't want Cloud Agent doing archive work (smart default), restrict the `ios-release` agent's `target:` to `vscode` so it never auto-routes in cloud.

### Pitfall 21: Test isolation broken by shared CoreData store

XCTests that share a CoreData store between tests have flaky behavior — test A leaves data, test B reads it, snapshot mismatch. Cmd+U passes locally; CI flaps.

**Right answer:** instruction file `coredata.instructions.md` Hard rule: *"Each test case creates a fresh `NSPersistentContainer` with `inMemory: true` (URL = /dev/null), isolated from app store. `setUp()` instantiates; `tearDown()` releases. Never use `UIApplication.shared.delegate.persistentContainer` in a test."*

### Pitfall 22: Snapshot tests with non-deterministic content

Snapshot tests that include current timestamp, random IDs, or device-specific UI (Dynamic Island vs notch) flake constantly. Each run produces a different snapshot.

**Right answer:** instruction file (e.g. `tests.instructions.md`) Hard rule: *"Snapshot test fixtures use frozen dates (`Date(timeIntervalSince1970: 0)`), seeded RNGs, and the `iPhone 14 Pro` simulator (consistent Dynamic Island). Wrap the system under test in a snapshot-time provider."*

## Cross-links

- [`01-PRINCIPLES.md`](01-PRINCIPLES.md) — the seven principles these pitfalls reinforce
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — which specialist owns each pitfall's domain
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — how to encode these as path-globbed rules
- [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) — what's enforced vs documentation-only
