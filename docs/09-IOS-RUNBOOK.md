# 09 — iOS Bootstrap Runbook

Step-by-step iOS bootstrap. ~2-4 hours total. You can split across multiple sittings.

## Phase 0 — Prerequisites (~10 minutes)

1. **Copilot signed in.** Verify in Xcode (or VS Code with Swift Language Server, or JetBrains AppCode if you still use it).
2. **Git clean.** Existing repo state should be clean before bootstrapping (otherwise you can't tell what bootstrap added vs what was already pending).
3. **Test build passes.** Run `xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build` once before starting. If it fails, fix that first — the framework can't help if the project doesn't compile.
4. **Decide your branching strategy.** Most iOS teams use `develop` for daily, `main` for releases, feature branches off `develop`. Confirm yours and have it ready for the bootstrap prompt.
5. **(Optional) Check MCP availability.** If you want MCP integrations, see [`11-IOS-MCP-CATALOG.md`](11-IOS-MCP-CATALOG.md). The framework works with zero MCPs; MCPs are amplifiers.

## Phase 1 — Inventory (read-only, ~30 minutes)

```bash
cd ~/path/to/MyiOSApp
git checkout -b setup/copilot-ios-framework
# (the bootstrap will create new files; you want a clean baseline)
```

Open Copilot Chat. Paste [`prompts/INVENTORY-PROMPT.md`](../prompts/INVENTORY-PROMPT.md) verbatim. Replace `<framework path>` with the absolute path to this repo on your machine.

The INVENTORY prompt is **read-only**. It scans:
- Existing `.github/copilot-instructions.md` if any
- Existing `.github/instructions/`, `.github/prompts/`, `.github/agents/`
- iOS-specific signals: `.xcworkspace`, `*.xcodeproj`, `Podfile`, `Package.swift`, `fastlane/`, `Tuist/`, etc.
- Tier 1 / Tier 2 docs already present

Then it proposes:
- Project name and orchestrator slug
- Recommended specialist list (the default 8, minus what doesn't apply, plus what your project specifically needs)
- Recommended `applyTo:` globs based on actual file paths
- Open questions you must answer before bootstrap

Review the proposal. Edit it. Answer the open questions in chat. **Do not proceed to bootstrap until the proposal makes sense for your project.**

## Phase 2 — Bootstrap (file generation, ~45 minutes)

In the same Chat session, paste [`prompts/BOOTSTRAP-PROMPT.md`](../prompts/BOOTSTRAP-PROMPT.md). It runs **mandatory pre-flight safety checks** before writing any file:

1. **Pre-flight 1** — auto-snapshot existing `.github/` to `.github-pre-bootstrap-backup/`
2. **Pre-flight 2** — naming-collision check per file; STOP for explicit user decision per conflict
3. **Pre-flight 3** — `applyTo:` glob conflict check; flag overlapping globs that would create contradictory rules
4. **Pre-flight 4** — drift detection on existing `copilot-instructions.md`; show a 3-pane diff before merging
5. **Pre-flight 5** — existing agent / chatmode style detection; surface parallel systems for explicit migrate-vs-coexist decisions
6. **Decision gate** — STOP if any pre-flight raised a `<NEEDS USER CONFIRMATION>` flag

After confirmation (per file), bootstrap generates:
- `.github/copilot-instructions.md` (thin router, customized for your iOS app)
- `.github/agents/<your-app>-orchestrator.agent.md` (orchestrator persona, with the `agents:` allowlist)
- `.github/agents/ios-ui.agent.md`, `ios-data.agent.md`, `ios-network.agent.md`, `ios-tests.agent.md`, `ios-release.agent.md`, `ios-privacy.agent.md`, `ios-perf.agent.md`, `ios-bg.agent.md` (the iOS specialist roster — selectively, per your inventory choices)
- `.github/instructions/swiftui.instructions.md`, `concurrency.instructions.md`, `coredata.instructions.md`, `networking.instructions.md`, `info-plist.instructions.md`, `code-signing.instructions.md` (path-globbed rules)
- `.github/skills/commit-push-pr/SKILL.md`, `verify-build/SKILL.md`, `correction-capture/SKILL.md`, `<your-app>-engineering/SKILL.md` (agent skills, iOS-pre-filled — cross-surface; the IDE-only prompt-file forms are optional)
- `docs/ai-context/INDEX.md`, `HANDOFF_SCHEMA.md`, `ORCHESTRATION_SPOONFEEDER.md` (orientation maps)
- `docs/ai-context/PROJECT.md`, `LEARNINGS.md`, `GLOSSARY.md` + one `docs/<AREA>_BACKLOG.md` (the project-truth set — [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md))
- `docs/_archive/README.md` (frozen archive marker)
- (Later phase, opt-in) `.github/hooks/framework.json` + `.github/scripts/*.mjs` — [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md)

Each file shows you a preview before writing.

## Phase 3 — Verify (15 minutes)

```bash
# 1. List what was created
ls -la .github/agents/ .github/instructions/ .github/skills/ docs/ai-context/

# 2. Verify the build still passes (the framework is doc-only; build should be unaffected)
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build

# 3. Sanity-check the generated content
cat .github/copilot-instructions.md           # should be thin (under 200 lines)
cat .github/agents/<your-app>-orchestrator.md # should mention your project name
cat .github/instructions/swiftui.instructions.md  # should reference your actual paths
```

## Phase 4 — First task end-to-end (30 minutes)

Open Copilot Chat. Type `@<your-app>-orchestrator`. Pick a real but small task — e.g. *"add a unit test for the existing `User.streak` computed property in `UserModelTests.swift`"*.

Watch the orchestrator:
1. Issue a `handoff:` block to `ios-tests`
2. `ios-tests` auto-loads `tests.instructions.md` (or whatever your test instruction file is)
3. `ios-tests` writes the test, runs `xcodebuild test`, returns `verified_claims` with the test name + pass result
4. Orchestrator validates the return, reports done

If anything looks wrong (orchestrator skips the handoff format, specialist doesn't read the instruction file, return is vague) — that's where the framework needs tightening. Add to your `docs/ai-context/INDEX.md` under "Known issues" and address in REFINEMENT.

## Phase 5 — Commit + PR (15 minutes)

```bash
git add .github/ docs/ai-context/ docs/_archive/
# .github-pre-bootstrap-backup/ is gitignored by the framework's .gitignore template
git commit -m "chore: bootstrap Copilot iOS orchestration framework"
git push -u origin setup/copilot-ios-framework
gh pr create --base develop --title "Bootstrap Copilot iOS orchestration framework"
```

Reviewer checklist:
- `.github/copilot-instructions.md` reads as a thin router (under 200 lines)
- Each specialist has a sensible `tools:` allowlist (REVIEW-ONLY agents have NO Edit/Write)
- Each instruction file's `applyTo:` glob actually matches files in the project
- `docs/ai-context/INDEX.md` has clean routing tables
- `xcodebuild` still passes

Merge.

## Phase 6 — Daily use (the next 2-3 weeks)

For 2-3 weeks, use the framework on real tasks. Pay attention to:
- Which specialists fire? Which never get used?
- Which instruction files auto-load when expected? Which never seem to?
- What does the orchestrator get confused about?
- What pitfalls are missing from your `docs/ai-context/`?
- What corrections do you keep issuing?

Keep a notes file (e.g. `docs/_archive/<this-month>/bootstrap-feedback.md` is fine for now). Don't fix things in real-time — log and let them aggregate.

## Phase 7 — Refinement (30 minutes)

After 2-3 weeks of real use, paste [`prompts/REFINEMENT-PROMPT.md`](../prompts/REFINEMENT-PROMPT.md). It walks you through:
- Audit specialist scope (drop unused ones, refactor over-broad ones)
- Audit instruction-file `applyTo:` (tighten globs, fix non-matching ones)
- Audit `copilot-instructions.md` length (it crept past 200 lines? trim)
- Audit `docs/ai-context/` for tier-1 → tier-3 archive moves
- Add new pitfalls from your notes file
- Add new instruction files for newly-discovered domains
- Re-check platform drift (Pitfall 23) and re-stamp `PROJECT.md` §3 (what is live on the App Store / TestFlight)

Output: a PR with the refinement changes. Merge.

## Phase 8 — Steady state (per-quarter)

Run REFINEMENT once per quarter (or per major iOS release). Add new pitfalls. Archive stale docs. Tighten globs.

The framework is now part of your project's invisible infrastructure.

## What the timeline actually looks like

| Phase | Time | When |
|---|---|---|
| 0 — Prerequisites | 10 min | One-time |
| 1 — Inventory | 30 min | One-time |
| 2 — Bootstrap | 45 min | One-time |
| 3 — Verify | 15 min | One-time |
| 4 — First task | 30 min | Sanity-check |
| 5 — Commit + PR | 15 min | One-time |
| 6 — Daily use | 2-3 weeks (passive) | Ongoing |
| 7 — Refinement | 30 min | After daily use |
| 8 — Steady state | 30 min/quarter | Ongoing |

Total active investment to fully bootstrap: **~2.5 hours** (Phase 1-5) + **30 min refinement** after 2-3 weeks. Total elapsed time to "fully cooked" framework: ~3 weeks.

## Cross-links

- [`prompts/INVENTORY-PROMPT.md`](../prompts/INVENTORY-PROMPT.md), [`BOOTSTRAP-PROMPT.md`](../prompts/BOOTSTRAP-PROMPT.md), [`REFINEMENT-PROMPT.md`](../prompts/REFINEMENT-PROMPT.md)
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — what each specialist looks like
- [`08-IOS-COMMON-PITFALLS.md`](08-IOS-COMMON-PITFALLS.md) — what to watch for during real use
- [`11-IOS-MCP-CATALOG.md`](11-IOS-MCP-CATALOG.md) — optional MCP integrations to add
- [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md) — the project-truth set bootstrap generates
- [`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md) — when the app and its API repos need one orchestrator above them
