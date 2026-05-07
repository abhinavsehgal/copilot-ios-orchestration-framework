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
3. Check the file's body length — is it within the ~3,000 char code-review cap?
4. Check for overlapping globs across instruction files

Report:
- **Healthy globs** (matches real files, under cap, no overlap)
- **Stale globs** (matches no files — project layout changed; propose new glob or removal)
- **Overweight files** (> 3,000 chars; propose split)
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

### A8. Slash prompt audit

For each `.github/prompts/*.prompt.md`:
- Is it being invoked? How often?
- Does the workflow still match the team's actual git / CI / release flow?
- Any new repeatable workflow worth a new slash prompt?

Propose additions / modifications / removals.

## Phase B — Propose changes (NO writes yet)

Output the audit as a single proposal block:

### Proposed changes

| # | Change | File(s) | Reason | Risk |
|---|---|---|---|---|
| 1 | Remove `ios-watch` specialist | `.github/agents/ios-watch.md` | 0 invocations in 90 days; project doesn't ship watchOS | Low |
| 2 | Tighten `swiftui.instructions.md` `applyTo:` from `Sources/**/*.swift` to `Sources/Views/**/*.swift,Sources/**/*View.swift` | `.github/instructions/swiftui.instructions.md` | Current glob over-fires on non-view files, polluting context | Low |
| 3 | Split `swiftui.instructions.md` (currently 4,200 chars) into rendering + state + accessibility | `.github/instructions/swiftui-*.instructions.md` (3 files) | Over the 3k char code-review cap; review never sees the back half | Med |
| 4 | Add Hard rule to `concurrency.instructions.md`: *"async functions returning to UI use `await MainActor.run { }` or `Task { @MainActor in }`"* | `.github/instructions/concurrency.instructions.md` | 4 PR review hits this month for missing main-thread switch | Low |
| 5 | Archive `docs/ai-context/visionos-companion-plan.md` | `docs/_archive/<YYYY-MM>/` | We decided not to ship visionOS; doc is now historical | Low |
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
   git commit -m "chore: refinement pass — <count> changes (specialists, globs, instructions)"
   ```
2. Suggest opening a PR with the audit summary as the description so reviewers see the rationale

## Hard NOs

- **NEVER** apply changes without explicit user approval per change (or explicit "apply all")
- **NEVER** modify `Sources/`, `Tests/`, or any user code
- **NEVER** delete files outside `.github/` or `docs/ai-context/` — only archive moves are allowed; tier-2 docs and source code are off-limits
- **NEVER** overwrite a file that has uncommitted local changes — show the user, ask whether to discard or stash

Begin Phase A audit now. Output the full audit before proposing any changes.
