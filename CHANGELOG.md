# Changelog

All notable changes to the Copilot iOS Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.1.1] — 2026-08-22

### Fixed — same-day audit of v1.1.0 (one runtime defect, two name mismatches, stale idioms)

- **Agent templates used Claude Code tool names** (`Read, Edit, MultiEdit, Bash, Grep, Glob, mcp__x__*`) in `tools:` — Copilot ignores unknown names, so the REVIEW-ONLY "runtime lock" enforced nothing. Mapped to the documented Copilot aliases (`read/edit/search/execute`, `<server>/*`); REVIEW-ONLY agents no longer carry `bash`.
- **Specialists were named `<PROJECT_SLUG>-ios-ui` in every template but `ios-ui` in every doc** (100 occurrences) — templates now match the docs and chapter 13's collision argument.
- Five of the nine iOS pitfalls (34, 35, 37, 39, 40) still carried cross-platform-framework idioms — restated as native iOS facts (`authorizationStatus`, `WKNavigationDelegate`, `URLSession`/`HTTPCookieStorage`/`cachePolicy`, scene-delegate handler re-runs, `strings` limits).
- Fastlane/Match presented as the only release/signing path → "Fastlane or Xcode Cloud" (+ `ci_scripts/` glob); `MyApp.xcworkspace` literals → `<WORKSPACE>`/`<SCHEME>` with the `.xcodeproj` form; `--silent` (not an `xcodebuild` flag) → `-quiet`; Xcode/AppCode no longer presented as agent drivers (Copilot-for-Xcode custom agents: not verified); bare `.agent.md` naming (17 sites); a nonexistent `docs/ai-context/HOOKS.md` reference; `.github/agents/<NAME>.md`; "3,000 chars" split threshold.
- Quickstart: "five tools" vs "six"; example orchestrator names aligned with chapter 13; "every repo ships a `backend-api`" (false for the app repo). Chapter 13 layout no longer lists a workspace hooks file with no template.
- `doc-freshness-track.mjs` gained Rule 12 (a push in another repository is not this repo's push) and `<REPO_NAME_FRAGMENT>`.

### Added

- **`templates/instructions/tests.instructions.md.template`** — XCTest/XCUITest/snapshot invariants (per-test stores, deterministic snapshots pinned to a destination, accessibility identifiers, protocol-backed doubles); `ios-tests` and the runbook now point at it.
- **`templates/workspace/bootstrap.sh.template`** — creates the whole workspace layer from a filled `workspace.json` (copies, placeholders, `.gitignore`, `.code-workspace` folders, `agents:` allowlist, service-map rows). Quickstart Part 3 step 2 uses it.
- BOOTSTRAP placeholder map explaining every `<TOKEN>` (incl. the `<XCODE_WORKSPACE>` vs `<WORKSPACE>` convention); REFINEMENT Phase B tied to audit items.

### Still open

- No field trial on a real iOS codebase yet. No `ci.instructions.md` template for Xcode Cloud `ci_scripts/`.

---

## [1.1.0] — 2026-08-22

### Retracted — the platform moved (verified against docs.github.com + code.visualstudio.com, 2026-08-22)

Three claims this edition made in v1.0.0 are no longer true, and one number is gone:

- **"Copilot has no programmable lifecycle hooks."** It does: `.github/hooks/*.json` with `sessionStart`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop`, `subagentStart/Stop`, `errorOccurred`, `preCompact`, `permissionRequest`, `notification` — on the cloud agent (reads only `.github/hooks/*.json`), the Copilot CLI and VS Code (which also reads a Claude-style `.claude/settings.json`). `preToolUse` returns `permissionDecision` (fail-closed on non-zero exit); `agentStop` can return `{"decision":"block"}` and receives `stop_hook_active`; timeouts fail open. **Chapter 10 is rewritten** around this contract, with an `xcodebuild` build-gate sized for a busy CI box (20-minute internal cap under a 25-minute hook timeout — the hook timeout must stay above the cap, because a timeout silently allows the stop). Pitfall 19 is retracted and rewritten.
- **"Cross-agent invocation has no allowlist."** VS Code's `agents:` frontmatter *is* an allowlist of subagents (`*` = all), invoked via `runSubagent`, with nesting behind `chat.subagents.allowInvocationsFromSubagents`; GitHub's `agent` tool alias exists on the cloud agent (no allowlist there). Chapter 3 and Principle 5 corrected; the orchestrator template carries `agents:`.
- **"Prompt files are where repeatable workflows live."** Agent skills (`.github/skills/<name>/SKILL.md`, also `.claude/skills/`) are the documented cross-surface primitive — cloud agent, code review, CLI, VS Code, JetBrains. Prompt files are IDE-only, and VS Code says to convert a prompt to a skill for the Agent Host. Chapters 2, 5, 6 and the README corrected; `/commit-push-pr`, `/verify-build` and `/correction-capture` now ship as skills with the same `xcodebuild` pre-fills.
- The "code review reads only the first 4,000 characters" cap is no longer on the current docs; current guidance is "about 1,000 lines per instruction file" / "no longer than two pages". Pitfall 5 rewritten; chapters 2, 5, 6, 10 and the `swiftui` instruction template corrected. Chat modes are retired (rename `.chatmode.md` → `.agent.md`); agent files use the `.agent.md` suffix.

### Added — what three more months of production use taught (ported from the stack-agnostic edition v1.2.0, iOS-flavoured)

- **`docs/12-PROJECT-TRUTH-AND-LEARNINGS.md`** — three knowledge stores a fresh agent reads first (`PROJECT.md` with a date-stamped "what is live where" table across App Store / TestFlight / internal builds / backend; `LEARNINGS.md` with decisions / failed approaches / bug patterns / agent corrections; per-area backlogs), "deferred work must be written, not spoken", "every production push — or App Store submission, or external-TestFlight promotion — freshens the docs in the same turn", the six-gate engineering playbook, the evidence-confidence taxonomy, the proof ladder, and the multi-client parity rules (the iOS app is usually a consumer of API repos it does not own).
- **`docs/13-MULTI-REPO-WORKSPACES.md`** — one iOS app repo + N backend/API repos (+ maybe Android) in separate repos: three layers (per-repo install → org-level agents → workspace repo with a `.code-workspace`, a manifest and gitignored clones, no CI), the verified per-surface table of what loads from several folders, three delegation mechanisms (the child's own `copilot -p` session; VS Code subagents; one cloud-agent task per repo), the name-collision hazard and how the design avoids it, two additive handoff fields, the cross-repo contract protocol, **shared Swift packages as contracts (pin by version tag, never by branch, in `Package.swift`)**, and a one-afternoon POC recipe ("add `estimatedDelivery` to the orders API response and show it on the iOS order screen"). Answers "do we need a fourth framework?" — no.
- **Pitfalls 23–31 (framework):** the platform moves under your conventions (re-verify quarterly); an instruction file is a claim, not evidence; deferred work in prose vanishes; production push without doc freshening; correction-regex false positives; a killed `xcodebuild` is inconclusive, not failed (and Copilot hook timeouts fail open); many sessions on one working directory (DerivedData and `-derivedDataPath` collide across concurrent sessions); never report a negative from a capped reader; two words for one thing ships bugs.
- **Pitfalls 32–40 (iOS-specific):** the APNs environment is decided by the entitlement, not by the build configuration; `apns-topic` must be the bundle id of the app that minted the token; iOS grants exactly ONE notification-permission prompt per install; a `WKWebView` cannot see a single-page app's route changes through its navigation delegate; offering a third-party login puts you outside Guideline 4.8's exception; `NSURLSession` keeps a cookie jar you did not choose — and `NSURLCache` masks sign-out; opposing orientation locks within ~1.5 s can jam the transition; a "last notification response" API returns the same tap forever; a build flavour changes the bundle id and the environment — not the source branch.
- **Templates:** `PROJECT.md` (iOS environments + what CI does NOT do — profile rotation, App Store Connect metadata, TestFlight promotion), `LEARNINGS.md`, `BACKLOG.md`, `GLOSSARY.md`, `engineering-playbook-skill.md` (as an agent skill); `hooks/` (`hooks.json`, `hook-io.mjs`, `correction-detect.mjs`, `doc-freshness-track.mjs`, `lint-fix.mjs` pre-filled for `swiftformat` on `*.swift`, `stop-gate.mjs` pre-filled with the `xcodebuild -workspace <WORKSPACE>.xcworkspace -scheme <SCHEME> -destination 'generic/platform=iOS Simulator' build` gate, the `Sources|Tests` build-relevant regex and a 20-minute cap — Patterns 2–5 on the real Copilot contract, flag-file design so no undocumented transcript format is parsed; design Rule 12 — a push in another repository is not this repo's push — added the same day); `skills/` (`commit-push-pr`, `verify-build` — converted from the v1.0 prompt files, `xcodebuild` pre-fills kept — and `correction-capture`); `workspace/` (the whole layer 3: `.code-workspace`, `workspace.json` with `ios-app` / `orders-api` examples, router, orchestrator + `contract-guardian` + `service-mapper` agents, `cross-repo-contracts.instructions.md`, `/delegate` skill, `sync-repos.sh`, `delegate.sh`, service map, contracts doc).
- **Chapter 4:** optional additive fields `repo`, `contract_impact`, `contracts_changed`, `deferred_work`; evidence-confidence class on claims. `schema_version` stays 1.
- **Chapter 6:** the agentic Copilot CLI (`copilot -p`, `--agent=`, `--allow-tool=`, run from the repo directory — no `--cwd`) as a mode; the surface matrix with Skills and Hooks rows.
- **Prompts:** BOOTSTRAP generates the project-truth set and the skills; INVENTORY scans skills, hooks and the project-truth set; REFINEMENT gains §A9 platform drift and §A10 project-truth freshness.
- **Root router template** gains golden rules 9–12 (corrections → instruction files; deferred work written; production push — App Store / TestFlight included — freshens docs, `agentStop`-enforced if hooks are installed; one name per concept) and the "read `PROJECT.md` §3 + skim `LEARNINGS.md` §D" workflow step.

### Changed

- `templates/agents/*.md.template` — file name `.agent.md` (noted atop every template); the orchestrator template carries `agents: [<SPECIALIST_LIST>]` (VS Code subagent allowlist; ignored by the cloud agent).
- Chapter 3 — specialists still return `recommended_next_agent` rather than chaining: a framework convention for auditability now that nesting is possible, not a platform limit.
- Chapter 2 — `.github/skills/` and `.github/hooks/` in the tree; `.github/chatmodes/` marked RETIRED; a "Multiple repositories" pointer to chapter 13.
- `templates/prompts/*.prompt.md.template` — kept as the IDE-only forms, each pointing at the skill that supersedes it; `chatmode.md.template` marked RETIRED.
- README — v1.1.0 banner; "What's in the box" with chapters 12–13 and the new template directories; surfaces list corrected (skills, hooks, `.agent.md`, chat modes retired); FAQ gains "prompt files or skills?" and the multi-repo question; companion section notes the joint 2026-08-22 release.

### Known gap

- **Still no proven adoption on a real iOS codebase** — the multi-repo layer in particular is designed from verified platform behaviour, not from an iOS field trial. The nine iOS-specific pitfalls (32–40) are the exception: each was learned on a shipping app.

### Provenance

The framework lessons (chapters 12–13, Pitfalls 23–31, the hook patterns, the project-truth set) were learned on the same production codebase that runs the stack-agnostic Copilot edition and the Claude Code edition side by side, and are ported here verbatim where they are domain-agnostic and iOS-flavoured where an example needed a concrete shape. Every platform claim was re-verified on 2026-08-22 and carries that date, because Pitfall 23 is the lesson that made this release necessary. Everything project-specific was stripped: if a lesson could not be restated as "any iOS team hits this", it stayed out. All three editions — Claude v1.2.0, Copilot v1.2.0, iOS v1.1.0 — released together on 2026-08-22.

---

## [1.0.0] — 2026-05-07

### Initial public release

A standalone, iOS-focused fork of the architecture established in [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) v1.1.2.

### Added

- **Eleven-chapter doc set** covering the seven principles, architecture, iOS specialist roster, handoff schema with iOS examples, path-globbed instructions for iOS file layouts, invocation modes across IDEs + Cloud Agent, three-tier folder structure, twenty-plus iOS-specific common pitfalls, the iOS bootstrap runbook, mechanical enforcement (and why hooks don't apply on Copilot), and an iOS MCP integrations catalog.
- **Eleven iOS specialist agent templates** — orchestrator + `ios-ui` (SwiftUI / UIKit / accessibility) + `ios-data` (CoreData / SwiftData / Realm) + `ios-network` (URLSession / Combine / cert pinning) + `ios-tests` (XCTest / XCUITest / snapshots) + `ios-release` (Fastlane / TestFlight / ASC) + `ios-privacy` (REVIEW-ONLY: Info.plist / ATT / nutrition labels) + `ios-perf` (Instruments / hangs / launch time) + `ios-bg` (BGTaskScheduler / silent push) + generic review-only and specialist shapes for custom additions.
- **Six iOS instruction file templates** — SwiftUI lifecycle invariants, Swift concurrency rules, CoreData multi-context patterns, networking + cert pinning gotchas, Info.plist usage descriptions, code signing + provisioning rotation rules.
- **Three iOS-anchored bootstrap prompts** — INVENTORY (read-only iOS project scan), BOOTSTRAP (file generation with iOS pre-flight safety checks), REFINEMENT (post-bootstrap iOS audit).
- **Three iOS-pre-filled slash prompt templates** — `/correction-capture` (iOS-flavored), `/commit-push-pr` (xcodebuild + iOS git workflow pre-filled), `/verify-build` (xcodebuild + scheme detection pre-filled).
- **Curated iOS MCP integration catalog** at [`docs/11-IOS-MCP-CATALOG.md`](docs/11-IOS-MCP-CATALOG.md) — Xcode + simulator MCPs, mobile UI automation MCPs, App Store Connect API MCPs, Sentry / Firebase / Crashlytics MCPs, GitHub MCP, Slack MCP. Each entry has a "what it's for / when to install / minimum tier" rubric.
- **Pre-flight safety pass** for brownfield bootstrap — snapshot existing `.github/copilot-instructions.md`, naming-collision detection, `applyTo:` glob conflict detection, drift detection, decision gate before any file write.

### Provenance

The principles, handoff schema, three-tier docs structure, and bootstrap workflow originated in the parent stack-agnostic framework. This iOS framework specializes them: every example, every specialist scope, every path glob, every pitfall, every MCP recommendation is iOS-flavored from the start. No mental translation required.

### Why a separate framework instead of an iOS section in the parent

The parent framework's strength is being stack-agnostic. Loading it with iOS-specific specialist names, build commands, and pitfalls would make it noisier for web / backend adopters. iOS engineers benefit from a focused framework that "just knows" the toolchain — `xcodebuild` instead of `npm run build`, `Info.plist` instead of `next.config.ts`, `App Store Connect` instead of `Stripe webhooks`.

### Known limitations

- **No proven adoption yet on a real iOS codebase.** This is v1.0; iOS adopters' first-week feedback drives v1.0.1 / v1.1.0.
- **MCP catalog is a snapshot.** The MCP ecosystem moves fast. Catalog will need refreshes; adopters should always check their MCP registry before installing.
- **No watchOS / tvOS / visionOS-specific specialist templates yet.** If your app has heavy companion-platform code, you'll add `ios-watch` / `ios-tv` / `ios-vision` specialists in your project — the templates support that pattern, but no pre-built templates ship in v1.0.

---

## Companion releases

- [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) — the stack-agnostic parent.
- [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) — the Claude Code equivalent.

This framework is on its **own version line** (v1.1.0 here alongside the parent's v1.2.0) — a separate maturation path. iOS-specific releases track the iOS toolchain (new Xcode versions, new App Store policies, new SwiftUI APIs) independently of the parent framework's cadence; platform-wide lessons (like the v1.1.0 hooks / skills / project-truth port) are released in lock-step across all three editions.
