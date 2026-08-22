# 08 — iOS Common Pitfalls

Forty hard-won lessons — framework / configuration pitfalls, iOS-specific pitfalls, and the
platform-drift and production lessons that v1.1.0 added. Read this before bootstrapping.

> Pitfalls 1–22 date from v1.0.0. Pitfalls 23–31 (framework) and 32–40 (iOS-specific) were added in
> v1.1.0; Pitfalls 5 and 19 were rewritten in v1.1.0 because the platform facts they rested on changed
> (verified against the official Copilot and VS Code docs on 2026-08-22), and Pitfall 23 records
> which v1.0 claims the platform has since made false.

## Framework / configuration pitfalls (1-7)

### Pitfall 1: Putting the orchestrator's persona in `.github/copilot-instructions.md`

The repository-wide instructions file is auto-loaded into EVERY Copilot interaction — including inline completions while you're typing in Xcode. Loading the orchestrator's full persona there:
- Slows down inline completions
- Pollutes code-review prompts with delegation instructions
- Forces every interaction through the orchestrator's "delegate everything" body

**Right answer:** Keep `.github/copilot-instructions.md` thin (golden rules + workflow + cross-links). The orchestrator agent is invoked explicitly via `@<your-app>-orchestrator`.

### Pitfall 2: REVIEW-ONLY agents with auto-Edit privileges

If you set `tools: read, search, grep, glob, edit` on `ios-privacy`, your privacy reviewer can now silently change `Info.plist`. The whole point of REVIEW-ONLY is the runtime lock — Copilot's harness physically blocks Edit if Edit isn't in `tools:`.

**Right answer:** REVIEW-ONLY agents have only `read`, `search`, `grep`, `glob`. Never include `edit` — and not `bash`/`execute` either, because a shell can write. (Use the tool aliases the Copilot docs list; an unrecognised name such as a Claude Code tool name is silently ignored, not enforced.)

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

### Pitfall 5: Instruction files have a documented size budget — long files get ignored or skimmed

The current GitHub docs give two size limits for instructions (verified 2026-08-22): the code-review tutorial says to **limit any single instruction file to about 1,000 lines**, and the repository-instructions page says **instructions must be no longer than 2 pages**. v1.0 of this framework cited a "code review reads only the first 4,000 characters" rule; that sentence is **no longer on the current docs** and should not be quoted as a hard cap.

The practical failure is unchanged: a long `swiftui.instructions.md` dilutes the rules that matter, and whichever reader is under the tightest budget (code review, inline completions in Xcode) sees the least of it. If your most-violated rule is on line 200, assume code review never weighs it.

**Right answer:** keep each `.github/instructions/<NAME>.instructions.md` well inside the documented budget (~2 pages; never anywhere near 1,000 lines) — split `swiftui` into rendering / state / accessibility files with their own `applyTo:` globs when it grows. Front-load the rules that matter most for code review (good practice, not a hard cap). Use `excludeAgent: "code-review"` for implementation-only guidance. Re-check the two size sentences quarterly (Pitfall 23); they have already changed once.

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

**Right answer:** `.gitignore` includes `*.cer`, `*.p12`, `*.mobileprovision`, `Profiles/*`, `keychain-*`, `*.keychain-db`. The `/commit-push-pr` skill explicitly refuses to stage any path matching those patterns. If discovered after the fact: rotate certs (treat as compromised), use git-filter-repo or BFG to scrub history, force-push, coordinate with team.

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

### Pitfall 19: Assuming Copilot has no hooks — or assuming every surface loads them the same way

**Retraction.** The v1.0 text of this pitfall said "Copilot does not expose programmable lifecycle events" and told you to wire automation into the IDE or a pre-commit hook instead. That is **no longer true** (verified 2026-08-22). Copilot now has lifecycle hooks — `sessionStart`, `sessionEnd`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop`, `subagentStart`, `subagentStop`, `errorOccurred`, `preCompact`, `permissionRequest`, `notification` — configured as `.github/hooks/*.json` (plus `~/.copilot/hooks/` for personal hooks). `preToolUse` can return `permissionDecision: allow | deny | ask` (exit 2 = deny, fail-closed); `agentStop` can return `{"decision":"block","reason":"…"}` to force another turn. Every correction-capture / build-gate / lint-fix / doc-freshness pattern the Claude edition runs as a hook now has a Copilot mapping — **[`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) carries the per-pattern translation, the hook I/O contract and the iOS-pre-filled templates (`xcodebuild` build-gate, `swiftformat` lint-fix)**. Don't rebuild them from this paragraph.

The pitfall that survives is subtler — three ways a hook you installed silently isn't running:

- **The cloud agent loads only `.github/hooks/*.json`.** VS Code additionally reads Claude-style hooks from `.claude/settings.json` (PascalCase event names), and the CLI reads `~/.copilot/hooks/`. A hook that lives only in `.claude/settings.json` works on a laptop and is absent from every cloud-agent run. Put team hooks in `.github/hooks/`.
- **Timeouts fail OPEN.** A hook that exceeds `timeoutSec` (default 30) is treated as if it had allowed the action. An `xcodebuild` build-gate that sizes its build to the timeout is a gate that opens exactly when the build is slow — see Pitfall 28 and size the gate's *own* internal cap (20 minutes in the template) below the hook timeout.
- **The platform-native half is still native.** `applyTo:` auto-loading on `.github/instructions/*.instructions.md` remains the right tool for rule surfacing (zero scripting, can't fail silently). A `preToolUse` hook that re-implements it is a second, weaker copy.

**Right answer:** read [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) before writing any hook. Install team hooks under `.github/hooks/`, verify each one fires on the surface you care about (cloud agent, CLI, VS Code are three separate checks), and treat a timeout as "inconclusive" in the hook's own logic rather than relying on the platform's fail-open default.

This pitfall most often hits teams running both editions of the framework: the documentation discipline (instruction files, agents, handoff schema) is shared, and the enforcement layer is now shared too — but the file locations and event names differ, so a hook ported by copy-paste lands in a directory one surface never reads.

### Pitfall 20: Cloud Agent without code signing access

The Cloud Agent runs in GitHub Actions. To run `xcodebuild archive` it needs signing certificates and provisioning profiles. By default it has neither.

**Right answer:** if you want Cloud Agent to do `archive` work, set up signing via [Match](https://docs.fastlane.tools/actions/match/) with secrets in GitHub Actions (or App Store Connect API key). Document in `docs/ai-context/release-process.md`. If you don't want Cloud Agent doing archive work (smart default), restrict the `ios-release` agent's `target:` to `vscode` so it never auto-routes in cloud.

### Pitfall 21: Test isolation broken by shared CoreData store

XCTests that share a CoreData store between tests have flaky behavior — test A leaves data, test B reads it, snapshot mismatch. Cmd+U passes locally; CI flaps.

**Right answer:** instruction file `coredata.instructions.md` Hard rule: *"Each test case creates a fresh `NSPersistentContainer` with `inMemory: true` (URL = /dev/null), isolated from app store. `setUp()` instantiates; `tearDown()` releases. Never use `UIApplication.shared.delegate.persistentContainer` in a test."*

### Pitfall 22: Snapshot tests with non-deterministic content

Snapshot tests that include current timestamp, random IDs, or device-specific UI (Dynamic Island vs notch) flake constantly. Each run produces a different snapshot.

**Right answer:** instruction file (e.g. `tests.instructions.md`) Hard rule: *"Snapshot test fixtures use frozen dates (`Date(timeIntervalSince1970: 0)`), seeded RNGs, and the `iPhone 14 Pro` simulator (consistent Dynamic Island). Wrap the system under test in a snapshot-time provider."*

## Framework pitfalls (23–31) — added in v1.1.0

> Ported from the stack-agnostic Copilot edition (its Pitfalls 20–28), where they were learned on a
> production codebase. Numbers differ because this edition already carried Pitfalls 20–22.

### Pitfall 23: The platform moves under your conventions — re-verify the docs every quarter

Three claims this framework made in v1.0 became false within a few months, and one number it
quoted disappeared from the docs entirely (all re-verified against docs.github.com and
code.visualstudio.com on 2026-08-22):

- **"Copilot has no hooks."** It does — `sessionStart` through `notification`, in
  `.github/hooks/*.json`, with `preToolUse` able to deny and `agentStop` able to block. The whole
  of [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) was rewritten; Pitfall 19 above is a retraction. Every
  correction-capture / build-gate / lint-fix / doc-freshness pattern now has a hook mapping.
- **"Cross-agent invocation has no allowlist."** In VS Code the `agents:` property on a custom agent
  *is* the allowlist for what it may invoke as a subagent (Chapter 3). The cloud agent still has
  none. The framework's *design* choice — specialists return `recommended_next_agent` rather than
  chaining — still stands, because a flat tree is auditable and a deep one is not; but it is a
  choice on VS Code now, not a platform gap.
- **"Prompt files are the Copilot equivalent of skills."** They are not. Agent Skills
  (`.github/skills/<name>/SKILL.md`) are the cross-surface primitive — cloud agent, code review,
  CLI, VS Code and JetBrains — and are the Copilot equivalent of Claude Code skills. Prompt files
  are an IDE-only convenience; the VS Code docs' own advice for a prompt file an agent won't use is
  "convert it to an agent skill". Chapters 2, 3, 5 and 6 were corrected.
- **"Code review reads only the first 4,000 characters."** That sentence is gone from the current
  docs. The documented limits are now "about 1,000 lines" per instruction file and "no longer than
  2 pages" for repository instructions (Pitfall 5). Front-loading stays good practice; it is no
  longer a cap.

Also changed, quietly: chat modes are retired (Chapter 2); the agentic `copilot` CLI replaced
`gh copilot suggest` as the headless surface (Chapter 6); VS Code reads `.claude/agents/`,
`.claude/rules/` and `.claude/settings.json` hooks by default.

**Right answer:** every framework claim about the platform carries a *verified-on* date. The
REFINEMENT prompt now includes a "platform drift" pass: re-read the custom-agents reference, the
Agent Skills page, the hooks reference and the Copilot CLI page, and diff them against Chapters 3,
5, 6 and 10. Treat a claim older than a quarter as `documented-unverified` (Chapter 12) until
re-checked. Where a Copilot equivalent of something is *not* on the current docs, say "not
documented" rather than guessing.

### Pitfall 24: An instruction file is a claim, not evidence

An instruction file documented the wrong HTTP verb for a route. Three agents trusted it, shipped a
client that called the wrong verb, and every user of that flow saw a generic "check your
connection" message for a week. The fourth agent read the route's `export` line.

Instruction files are written by people and agents at a point in time. They rot like any other
document — faster, because they are short and confident, and because `applyTo:` auto-loading puts
them in front of the agent with the authority of a system prompt.

**Right answer:** an instruction file is `documented-unverified` until you have looked at the thing
it describes. Before *relying* on a documented fact to build something, check it at the source (the
route, the schema, the config). When an instruction file is found wrong, fix it in the same turn —
never leave a known-false rule standing because "it's not my task".

### Pitfall 25: Deferred work that lives only in prose vanishes

"We'll do the retry logic in a follow-up" was said at the end of a session. No follow-up happened.
Two weeks later the missing retry was reported as a bug, investigated from scratch, and fixed — with
none of the context the first session had.

**Right answer:** a turn that names deferred work may not end until that work is appended to
`docs/<AREA>_BACKLOG.md` with what / why / effort / revisit-when (Chapter 12). The backlog id is
checked against the remote and open PRs first — two concurrent sessions took the same id on the same
day. Verbal follow-ups are not follow-ups.

### Pitfall 26: A production push that does not freshen the docs strands the next agent

A feature went to production on Tuesday. Its orientation map still described it as staging-only
behind a flag. On Thursday a fresh agent "enabled" it — and re-opened a migration that had already
been applied.

**Right answer:** every production push updates the full doc set *in the same turn*:
`docs/CHANGELOG.md` (the anchor), the affected orientation maps, the affected instruction files, and
`PROJECT.md` §3 if the production *state* changed. Documentation discipline missed this about one
push in five; it is now an `agentStop` hook (doc-freshness gate, Chapter 10 Pattern 5). A future
agent sees only what is in git.

### Pitfall 27: Correction-capture regexes false-fire on benign phrases — and the cost is trust

The correction-capture hook (Chapter 10, Pattern 2) fired on *"You already have access to it"* — a
perfectly polite sentence — because its regex matched `you already`. It also fired on a test plan
that *quoted* a correction phrase, and on a framework doc that *explained* the hook.

**Right answer:** three guards, all now in the template. (1) Anchor the frustration verbs to what
follows them (`you already (did|changed|broke)`), never bare. (2) Strip code fences, inline code and
heredoc bodies before matching — quoted text is data, not a correction. (3) Keep the loop guard
(`stop_hook_active`, which `agentStop` receives on its stdin payload) so a false fire costs one reply
("not a correction"), never a trapped session. When a false fire does happen, tighten the pattern
the same day; a hook that cries wolf gets disabled by the team within a week.

### Pitfall 28: A killed check is inconclusive, not failed

The build-gate `agentStop` hook capped `xcodebuild` at five minutes. On a CI box also running two
other builds — cold DerivedData, a Pods/SPM resolve — a legitimate twelve-minute build was killed
every time and reported as *failed*, with a warnings-only tail (the half-written log of a build
that never reached its first real error) the agent then tried to "fix". An alarm loop with nothing
to fix.

On Copilot there is a second timer to respect: the hook's own `timeoutSec` (default 30), and a hook
that exceeds it **fails OPEN by design** — the platform treats it as if it had allowed the stop.
So a build-gate has two bad shapes: one that reports a kill as a failure (the alarm loop), and one
whose build runs longer than the hook timeout (a gate that is open precisely when the build is
slow, with no signal that it opened).

**Right answer:** distinguish "killed by timeout or signal" from a non-zero exit inside the script.
A killed run has produced **no failure evidence**; exit 0 silently and let CI (which builds every PR
anyway) be the authority. Only a real non-zero exit blocks the stop. Size the gate's *own* internal
cap below the hook's `timeoutSec` (the template ships 20 minutes under a 25-minute hook timeout),
and size both to the slowest honest `xcodebuild` on a busy machine, not to the fastest one on an
idle machine.

### Pitfall 29: Many sessions, one working directory

Three agent sessions shared one checkout. One was launched into a stale, detached snapshot of the
tree and "fixed" code that had already been rewritten. Two ran `xcodebuild` at once against the
same DerivedData and the same `-derivedDataPath` / output directory and corrupted each other's
build products — the failure read like a code bug (missing module, mismatched `.swiftmodule`) and
was pure directory collision. Two picked the same migration number on the backend they both talked
to.

**Right answer:** `git fetch` and compare to the remote base *before the first edit* — the tree you
were launched in is not trustworthy. Do feature work in an isolated git worktree branched from the
fresh-fetched base — use separate git worktrees per session, and symlink the dependency directory
(`Pods/`, the SPM cache) rather than duplicating it. Give each concurrent session its own
`-derivedDataPath`, and never run two `xcodebuild` invocations against one output directory. Before
taking any sequential id (migration number, backlog id, schema version), check the remote *and*
other sessions' open PRs. (The Copilot CLI has no `--cwd` flag — run it *from* the worktree.)

### Pitfall 30: Never report a negative from a reader you have not verified can see the whole set

"The generator refused these 336 items" turned out to mean "the reader that listed the items was
silently capped at 1,000 rows and never showed the generator 25% of them." A confident negative
result, wrong, because the *reader* had a ceiling nobody checked.

Most data-access layers cap a bare query (ORMs, REST layers, search APIs — 1,000 is a common
default), and an explicit `limit` above the cap often does not raise it. No error, no truncation
flag.

**Right answer:** before reporting "none found", "it refused", or "zero exist", establish the reader's
ceiling. Page with a stable order, aggregate in the database, or use a count query. A capped read is
worse than a failed one: it looks like a confident answer.

### Pitfall 31: Two words for one thing ships bugs

The same concept was called `domain` in the database, `topic` on one component and `chapter` on the
page — and `topic` *already meant something else* in a legacy subsystem. A gate took the new
meaning, looked it up where the old meaning lived, matched nothing, and an empty match silently
widened a filter to the whole dataset. Every user of that gate got off-target results for a day.

Vocabulary drift is not cosmetic. It is how a correct-looking lookup returns the wrong set.

**Right answer:** one glossary (`docs/ai-context/GLOSSARY.md` — a rule file, not a nicety) naming
each concept exactly once, with the DB column, the type field and the UI label that carry it. Adding
a fifth name for an existing concept is a defect. Rename only while already editing the file that
carries the wrong name, and say so in the commit.

## iOS-specific pitfalls (32–40) — added in v1.1.0

> Each of these is a platform fact any iOS team can hit, learned the hard way on a shipping app.
> They are written to be copied into an instruction file (`.github/instructions/*.instructions.md`)
> with `applyTo:` globs matching the code they protect.

### Pitfall 32: The APNs environment is decided by the entitlement, not by the build configuration

A Release build signed with a **development** provisioning profile still has
`aps-environment = development`, so its device tokens are **sandbox** tokens. Sending them to the
production APNs host fails with `BadDeviceToken` — per device, silently, with nothing in the app
and nothing the user can see. Only a distribution-signed build (TestFlight / App Store) is
production.

**Right answer:** never infer the APNs environment from `DEBUG`, a scheme name or a build flavour.
Read the `aps-environment` entitlement at runtime and send it to the server with the token; the
server routes by what the token *says*, and skips a token whose environment it does not know
rather than guessing. Rule for `*.entitlements` and the push-registration code.

### Pitfall 33: `apns-topic` must be the bundle id of the app that minted the token

Teams that ship two variants of an app (production + QA / staging) on the same devices have two
bundle ids. A token minted by one bundle is rejected for the other with **400
`DeviceTokenNotForTopic`** — per device, silent, and easy to mistake for a dead token.

**Right answer:** the topic is a property of the *token*, not of the server. Store the minting
bundle id alongside each token (or, at minimum, configure the server-side topic per environment
and assert it matches the build that talks to it). Rule for any push-sending code and any place a
second bundle id is introduced.

### Pitfall 34: iOS grants exactly ONE notification-permission prompt per install

`requestAuthorization` shows the system dialog once. After a denial (`authorizationStatus ==
.denied`) it resolves instantly with the same answer and shows nothing
— a dead button. A prompt spent behind a rarely visited screen, or behind a registration call that
could not succeed, is a prompt lost for the life of the install.

**Right answer:** ask at a deliberate moment (onboarding / sign-in) and only once you have proven
the registration path works end to end; re-register silently when already granted; never call the
request API when the system says it cannot ask again — deep-link to Settings instead. Rule for the
permission / registration module.

### Pitfall 35: A `WKWebView` cannot see a single-page app's route changes through its navigation delegate

`WKNavigationDelegate` fires on document **loads**.
A client-side router navigates with `history.pushState` and re-renders — no load, no callback. A
native wrapper that watches for "the web page navigated away" is correct code waiting for an event
that never comes; the user ends up on an unintended web page wearing the app's chrome.

**Right answer:** inject a script at document start that patches `history.pushState` /
`replaceState` and listens for `popstate`, posting the path out via the message handler. Keep the
navigation delegate too — it catches the real loads (redirects, hand-off hops). Each covers a
half; neither covers both. Rule for any `WKWebView` host that depends on page navigation.

### Pitfall 36: Offering a third-party login puts you outside Guideline 4.8's exception

If the app offers any third-party sign-in (Google, Facebook, …), App Review expects an equivalent
privacy-preserving option — in practice Sign in with Apple. The exception for apps that
"exclusively use your company's own account setup" stops applying the moment the third-party
button is on the screen, and an email+password option does **not** satisfy the equivalence test
(it cannot hide the user's email). Adding Sign in with Apple is not a drop-in either: Hide My
Email returns a relay address, so any flow that treats the sign-in email as a *contact* address
for someone else (a guardian, an admin, an account owner) is now mailing a relay that forwards to
the person who signed in.

**Right answer:** decide at design time — own-accounts only (inside the exception) or third-party
+ Sign in with Apple (and audit every place the sign-in email is used as a contact). Keep a
one-line feature flag that removes the third-party button so a rejection can be answered in one
build. Rule for the authentication module and the privacy review checklist.

### Pitfall 37: `NSURLSession` keeps a cookie jar you did not choose — and `NSURLCache` masks sign-out

`URLSession` (and every HTTP client built on it) stores `Set-Cookie` from every response and silently attaches those
cookies to later requests to the same host. An app that authenticates with a bearer token can
therefore succeed *because of a cookie* left by a different role's earlier session — serving the
wrong person's data while every log says the token path works. `NSURLCache` compounds it: a
cached 200 keeps "working" after sign-out, hiding a real 401 for days.

**Right answer:** treat the cookie jar as a third credential. Use
`cachePolicy = .reloadIgnoringLocalCacheData` (or a session with no `URLCache`) on authenticated
calls, clear `HTTPCookieStorage` on sign-out, and when a request "works" do not conclude the
intended credential worked until you have reproduced it with a bare `curl` and only that
credential. Rule for the networking layer.

### Pitfall 38: Opposing orientation locks within ~1.5 s can jam the transition

Locking to landscape and then, while that rotation is still animating, locking back to portrait
froze the transition until the user swiped out. Orientation is one global switch, but every screen
instance that touches it does so independently, and instances overlap (leave → rejoin, duplicate
pushes).

**Right answer:** one owner of the orientation at a time (a module-level generation counter — only
the newest instance may restore on its way out), and a restore that is delayed until the entrance
lock has settled, re-checking ownership when the timer fires. Never add "re-assert" timers that
can race a legitimate exit. Rule for any screen that changes orientation.

### Pitfall 39: A "last notification response" API returns the same tap forever

APIs that answer "what was the last notification the user tapped" (a cached launch option, a
stored `UNNotificationResponse`, a third-party wrapper's "last response" getter) keep answering it.
A handler that reads it and navigates, re-run on every scene activation, view update or session
refresh, replays the morning's tap and tears out whatever screen the user just opened — with no
error and no fingerprint in any screen-level code, because the cause sits in the app / scene
delegate above every screen.

**Right answer:** consume the cold-start response exactly once per process (a module-level flag),
then clear it; never let a handler that can navigate re-run on every view update or state change.
Rule for the app / scene delegate and the notification-handling module.

### Pitfall 40: A build flavour changes the bundle id and the environment — not the source branch

A "QA build" that points at staging with a second bundle id is still compiled from whatever
branch is checked out. A tester on a QA build can be testing last week's code against today's
server, and a string search of the compiled binary will not prove otherwise (`strings` misses
non-ASCII and encoded literals, so a zero match proves nothing).

**Right answer:** stamp the git SHA into the build (`Info.plist` or an about screen) and read it
from the device before believing what a build contains. Rule for the release / CI configuration.

## Cross-links

- [`01-PRINCIPLES.md`](01-PRINCIPLES.md) — the seven principles these pitfalls reinforce
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — which specialist owns each pitfall's domain
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — how to encode these as path-globbed rules
- [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) — what's enforced vs documentation-only; the hook contract behind Pitfalls 19, 27 and 28
- [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md) — the stores and habits Pitfalls 24–26, 30 and 31 call for
- [`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md) — where the iOS app is one consumer among several
