# REFINEMENT-PROMPT (iOS post-bootstrap audit)

> **Paste this entire file into Copilot Chat 2-3 weeks AFTER bootstrap, or once per quarter as part of routine maintenance. The prompt audits your existing framework setup, identifies drift, and proposes fixes — but does NOT apply changes without your explicit approval per change.**

You are running a refinement pass on the Copilot iOS Orchestration Framework setup in this repository. Your job is to **audit the framework's actual usage** (not just its theoretical configuration) and propose tightening / cleanup / additions.

## Phase A — Read-only audit

### A1. Specialist usage audit

For each `.github/agents/*.md`:
- Has it been invoked in the last 30 days? (Heuristic: search PR titles + commit messages + issue threads for `@<agent-name>` mentions, and Copilot Chat history if accessible.)
- If invoked: how many times, on what kinds of tasks?
- If NOT invoked: is it defensively useful (e.g. `ios-privacy` REVIEW-ONLY agent that only fires on PRs touching Info.plist, low traffic but high impact), OR is it dead weight?

Report:
- **Hot specialists** (invoked > 5x in 30 days)
- **Warm specialists** (1-5x)
- **Cold specialists** (0x but defensible — keep them)
- **Dead specialists** (0x and not defensible — propose to remove)

### A2. Instruction-file `applyTo:` audit

For each `.github/instructions/*.instructions.md`:
1. Read the `applyTo:` glob
2. Run `find . -path '<glob>' | head -10` (or equivalent) — does it actually match files in the project today?
3. Check the file's body length — is it well inside the documented budget (about two pages / ~1,000 lines per file; the old "4,000 characters" cap is no longer on the docs)? The framework's own guidance is ≤ 150 lines.
4. Check for overlapping globs across instruction files

Report:
- **Healthy globs** (matches real files, inside the budget, no overlap)
- **Stale globs** (matches no files — project layout changed; propose new glob or removal)
- **Overweight files** (> 150 lines; propose split)
- **Overlapping globs** (two files would both auto-load on the same edits; propose consolidation)

### A3. `copilot-instructions.md` length audit

Read `.github/copilot-instructions.md`:
- Line count? (Target: under 200)
- Any orchestrator-persona-shaped content that should be in `.github/agents/<your-app>-orchestrator.md` instead?
- Any iOS-specific gotchas that should move to a dedicated `.github/instructions/<NAME>.instructions.md`?

Report each over-200-lines candidate for promotion / move.

### A4. `docs/ai-context/` tier-1 audit

For each `docs/ai-context/*.md`:
- Last-modified date — older than 6 months?
- Cross-links — any pointing to docs that no longer exist?
- Length — is the file still 50-150 lines, or has it grown to 300+? (Long orientation maps lose their value.)

Report each candidate for archive (move to `docs/_archive/<YYYY-MM>/`) or split.

### A5. `docs/<UPPERCASE>.md` tier-2 audit

For each canonical doc:
- Last update date
- Does it match current architecture? (Spot-check: pick one architectural claim and verify against current code)
- Any sections that are now historical and should move to archive?

Report each tier-2 doc with a "freshness" score (current / drifted / archive-candidate).

### A6. MCP allowlist audit

For each agent's `mcp-servers:` list:
- Is each MCP actually installed in the team's registry?
- Is each MCP actually used by the agent in practice?
- Any MCPs that should be added based on common workflows?

Report:
- **Phantom MCPs** in allowlists (listed but not installed; remove)
- **Unused MCPs** (installed and listed but agent never actually calls them; consider tightening)
- **Missing MCPs** (agent regularly does work that an installed MCP could accelerate)

### A7. New iOS pitfalls

In the last 30 days of PRs / issues / Crashlytics top issues / App Store rejection notes:
- Are there 2-3 recurring iOS-specific gotchas that should harden into `.github/instructions/<NAME>.instructions.md` Hard rules?
- Are there App Store / iOS-version-specific gotchas worth adding (e.g. iOS 18 introduced new privacy manifest requirements)?

Propose each as a one-line patch to the appropriate instruction file.

### A8. Skill + prompt-file audit

For each `.github/skills/*/SKILL.md` and each `.github/prompts/*.prompt.md`:
- Is it being invoked? How often? Is any skill auto-loading on unrelated tasks (vague `description`)?
- Does the workflow still match the team's actual git / CI / release flow (Fastlane lanes, scheme names)?
- Is any workflow the cloud agent or the CLI needs still a prompt file only (IDE-only)? → propose converting it to a skill
- A skill and a prompt file with the same name? → keep one
- Any new repeatable workflow worth a new skill?

Propose additions / modifications / removals.

### A9. Platform drift (v1.1.0)

The platform moves under the framework's conventions (Pitfall 23). Re-read the official pages for custom agents, Agent Skills, hooks and the Copilot CLI, and diff them against what this project's agents, instruction files, skills and `docs/ai-context/HOOKS.md` assume. Three claims this framework itself made in v1.0 are now retracted — check the project has not inherited them:

- "Copilot has no hooks" — it does (`.github/hooks/*.json`; `preToolUse` can deny, `agentStop` can block). Is the `xcodebuild` build-gate or correction-capture still living only in a pre-commit hook or an IDE setting because of the old claim?
- "Cross-agent invocation has no allowlist" — VS Code's `agents:` is one; the cloud agent still has none. Does the orchestrator declare `agents:` with the exact specialist names? Does any specialist (it should not)?
- "Prompt files are the Copilot equivalent of skills" — they are IDE-only; Agent Skills are cross-surface. Is any workflow the cloud agent or CLI needs still a prompt file only?

Also check: no `.chatmode.md` files remain (retired → `.agent.md`); agent files use the `.agent.md` suffix; the instruction-file size guidance quoted anywhere is the current one (about 2 pages / ~1,000 lines, not "4,000 characters"); any `copilot -p` delegation runs from the repo directory (no `--cwd`) and is granted the tools it needs; if hooks are installed, the `agentStop` `timeoutSec` is still above `stop-gate.mjs`'s `BUILD_CAP_MS` and both still exceed a cold `xcodebuild` on the slowest CI box. Report each stale assumption with the doc URL that contradicts it. Where the docs say nothing, write "not documented" — never guess.

### A10. Project-truth freshness (v1.1.0)

- `docs/ai-context/PROJECT.md` §3: is every row's environment state still true — App Store version/build, latest TestFlight build, backend per environment, flags? Diff against the changelog and App Store Connect since the header's verified-on date; re-stamp the header after fixing.
- `docs/ai-context/LEARNINGS.md`: any §D correction that has been violated in the last month despite being written down? That is the signal to promote it to a hook (Chapter 10).
- Backlogs: any `docs/*_BACKLOG.md` item older than a quarter with no revisit signal → propose archive or delete. Any "later" in recent PR descriptions that never reached a backlog?
- Glossary: any new name for an existing concept introduced since the last pass (a model field, an API field and a screen label for the same thing)?
- The engineering skill (`.github/skills/<project-slug>-engineering/SKILL.md`): is it still loading (appears under `/` in chat; used by the cloud agent)? Are reports still tagging claims with the confidence classes?
- If the app is a consumer of API repos in a workspace (Chapter 13): is `CONTRACTS.md` still true, and is every shared Swift package pinned by tag, not branch, in `Package.swift`?

## Phase B — Propose changes (NO writes yet)

Output the audit as a single proposal block:

### Proposed changes

| # | Change | File(s) | Reason | Risk |
|---|---|---|---|---|
| 1 | Remove `ios-watch` specialist | `.github/agents/ios-watch.md` | 0 invocations in 90 days; project doesn't ship watchOS | Low |
| 2 | Tighten `swiftui.instructions.md` `applyTo:` from `Sources/**/*.swift` to `Sources/Views/**/*.swift,Sources/**/*View.swift` | `.github/instructions/swiftui.instructions.md` | Current glob over-fires on non-view files, polluting context | Low |
| 3 | Split `swiftui.instructions.md` (currently 310 lines) into rendering + state + accessibility | `.github/instructions/swiftui-*.instructions.md` (3 files) | Well past the 150-line guidance; the rules that matter are buried | Med |
| 4 | Add Hard rule to `concurrency.instructions.md`: *"async functions returning to UI use `await MainActor.run { }` or `Task { @MainActor in }`"* | `.github/instructions/concurrency.instructions.md` | 4 PR review hits this month for missing main-thread switch | Low |
| 5 | Archive `docs/ai-context/visionos-companion-plan.md` | `docs/_archive/<YYYY-MM>/` | We decided not to ship visionOS; doc is now historical | Low |
| 6 | Convert `.github/prompts/verify-build.prompt.md` to `.github/skills/verify-build/SKILL.md` | `.github/skills/verify-build/SKILL.md` | The nightly `copilot -p` test triage cannot load a prompt file (IDE-only) | Low |
| 7 | Re-stamp `PROJECT.md` §3 — App Store is 4.2 (812), table says 4.1 | `docs/ai-context/PROJECT.md` | State claim older than the last release | Low |
| ... | ... | ... | ... | ... |

For each row, include:
- Clear before/after diff
- Specific file path
- One-sentence rationale
- Risk assessment (Low / Med / High)

## Phase C — Apply approved changes

User reviews the proposal and either:
- "Apply all" — go through each change and apply (showing diff per file before write)
- "Apply 1, 2, 4 — skip 3, 5" — apply selected
- "Discuss change 3 first" — drill into a specific change

For each applied change:
1. Show the diff
2. Apply the file change
3. Confirm post-write

After all approved changes are applied:
1. Suggest a commit:
   ```bash
   git add .github/ docs/ai-context/ docs/_archive/
   git commit -m "chore: refinement pass — <count> changes (specialists, globs, instructions, skills, project truth)"
   ```
2. Suggest opening a PR with the audit summary as the description so reviewers see the rationale

## Hard NOs

- **NEVER** apply changes without explicit user approval per change (or explicit "apply all")
- **NEVER** modify `Sources/`, `Tests/`, or any user code
- **NEVER** delete files outside `.github/` or `docs/ai-context/` — only archive moves are allowed; tier-2 docs and source code are off-limits
- **NEVER** overwrite a file that has uncommitted local changes — show the user, ask whether to discard or stash

Begin Phase A audit now. Output the full audit before proposing any changes.
