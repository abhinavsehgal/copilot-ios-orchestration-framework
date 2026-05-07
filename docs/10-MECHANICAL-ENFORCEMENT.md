# 10 — Mechanical Enforcement (and what to use instead)

GitHub Copilot does not expose programmable lifecycle hooks. There is no `PreToolUse` / `PostToolUse` / `Stop` event you can wire a script to. If you've come from another harness expecting hook-based enforcement, here's how the iOS framework achieves the same outcomes through different means.

## Translation table

| Hook pattern (other harnesses) | iOS Copilot equivalent | Quality |
|---|---|---|
| PreToolUse rule-surfacing | `applyTo:` frontmatter on `.github/instructions/<NAME>.instructions.md` | **Native and strict** — Copilot auto-loads matching files. Can't fail silently. |
| Stop correction-capture | `/correction-capture` slash prompt (manually invoked) | **Partial** — Copilot has no Stop event, so the prompt must be invoked explicitly after a correction. |
| Stop build-gate | `ios-release` agent's Definition of Done + `/verify-build` slash prompt | **Partial** — no automatic trigger. Enforcement lives in agent instructions ("the build must be green before claiming done") and a prompt the orchestrator runs as the last step. |
| PostToolUse lint-fix on edit | IDE auto-fix (Xcode "Format on Save", VS Code Swift format) + pre-commit hook (e.g. SwiftLint, SwiftFormat) | **Out of scope.** This belongs in IDE config or pre-commit, not in Copilot's surface. |

The takeaway: Copilot's design favors **declarative auto-loading** over imperative event hooks. Two of the four patterns translate cleanly; one is partial; one belongs elsewhere.

## Pattern 1 — Rule surfacing (already in this framework)

The most important hook pattern translates **natively** via `applyTo:` frontmatter. When Copilot is editing any file matching the glob, the instruction body auto-loads into context. Strictly better than a hook in one way: it can't fail silently — Copilot's harness handles it directly.

Recap (also in [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md)):

```yaml
---
applyTo: "Sources/Views/**/*.swift,Sources/**/*View.swift,*.xib,*.storyboard"
---

# SwiftUI / UIKit Instructions
[...rules...]
```

When a developer (or the autonomous Cloud Agent) edits any matching file, Copilot pulls the body in. Zero scripting. No hook to maintain. No bug class around "did the hook fire?"

Discipline you DO need:
- Keep instruction files ≤ 150 lines / ~3,000 chars
- Use `applyTo:` globs that actually match your file paths
- Front-load code-review-relevant rules (Code Review reads first ~4,000 chars only)
- Have a `context-librarian` specialist who audits `applyTo:` overlap quarterly

## Pattern 2 — Correction-capture (manual prompt)

In other harnesses, a Stop hook can detect strong correction signals in the user's message and force a rule patch. Copilot has no Stop event, so the equivalent is a **prompt the developer (or orchestrator) invokes explicitly after a correction**.

The framework ships [`templates/prompts/correction-capture.prompt.md.template`](../templates/prompts/correction-capture.prompt.md.template). After bootstrap it lives at `.github/prompts/correction-capture.prompt.md`. After any correction that feels like a recurring iOS pattern, type:

```
/correction-capture
```

The prompt walks Copilot through a 5-step checklist:
1. Is this a recurring iOS pattern (not a one-off)?
2. Find or create the right `.github/instructions/<file>.instructions.md`
3. Draft a one-line patch under "Hard rules" or "What NOT to do"
4. Show the proposed patch
5. Wait for user approval before applying

The discipline is "no `I'll remember`, only patches." Memory is not a substitute for an instruction file.

Trade-offs:
- ✅ No false positives (manual invocation only)
- ✅ Works the same on every Copilot surface
- ❌ Requires the developer to remember to invoke
- ❌ Easier to skip than a hook

To minimize the discipline cost, encode the `correction-capture` invocation step into your orchestrator agent's "incoming feedback" rules: *"if the user issues a correction, immediately run the `correction-capture` prompt before continuing."*

## Pattern 3 — Build-gate (Definition of Done)

Other harnesses can run `xcodebuild` automatically before letting an agent stop. Copilot has no Stop event, so this is enforced via:

1. **Agent Definition of Done** — every implementation specialist's body includes a rule like *"Before declaring `status: completed`, verify the build passes via `xcodebuild test ...` and include the command + result in `tests_run`."*
2. **Orchestrator return-validation** — the orchestrator rejects returns with empty `tests_run` for code changes.
3. **`/verify-build` slash prompt** — when in doubt, the orchestrator (or developer) invokes it. The prompt runs `xcodebuild build` (or your project's actual build command) and surfaces the failure tail.

The pattern relies on the orchestrator catching the violation on the return path rather than on a runtime trigger. Less mechanical than a Stop hook, but enforceable as long as the orchestrator's validation step is taken seriously.

iOS-specific build-gate considerations:
- Don't run `xcodebuild archive` in the slash prompt (slow + requires signing); use `xcodebuild build` for the gate
- For tests, `xcodebuild test -destination 'platform=iOS Simulator,name=iPhone 15'` (don't depend on physical-device flow in CI)
- For pure unit tests, `xcodebuild test -only-testing:<TestTarget>` is faster than building the full app

## What does NOT translate from other harnesses

### `PostToolUse` lint-fix has no Copilot equivalent

Copilot has no event for "after the file was just edited, run X." This belongs at the IDE layer:

- **Xcode:** Editor → Edit All in Scope, plus SwiftFormat / SwiftLint Build Phase script
- **VS Code:** Swift extension + `editor.formatOnSave` + `editor.codeActionsOnSave`
- **Pre-commit:** [pre-commit](https://pre-commit.com/) framework with [SwiftLint](https://github.com/realm/SwiftLint) + [SwiftFormat](https://github.com/nicklockwood/SwiftFormat) hooks

If you don't have an editor-level auto-fix yet, fix that before adopting any other parts of this chapter. Copilot will write code at whatever style your editor formats to; if your editor doesn't format on save, every PR will have formatting noise and Copilot has no way to fix it server-side.

### Hooks-on-Cloud-Agent

The Copilot Cloud Agent runs in GitHub Actions with no access to per-developer settings. Hooks-style runtime enforcement on the Cloud Agent would require GitHub Actions workflow steps — outside the framework's surface area. The framework's documentation discipline applies fully to the Cloud Agent because instruction files and prompt files are repository state, not local config.

For iOS specifically, this means: the Cloud Agent honors all your `.github/instructions/*.instructions.md` rules, all your agents' tool allowlists, and all your slash prompts. What it doesn't get is hook-based scaffolding that would exist on a Claude Code adopter's machine.

### Sidebar — Lesson from sister frameworks

If you also adopt [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) (the Claude Code companion) on a different stack, note: Claude Code's `Stop` hooks have a counterintuitive IO contract — `stdout` is captured but NOT surfaced to the model; only `stderr` is. The Claude framework's v1.1.0 shipped Stop hook templates that wrote reminders to stdout, which meant the reminders correctly produced + exit code was correct, but the model never saw them. v1.1.2 fixed it by switching to stderr.

This whole class of bug doesn't exist on the Copilot side because there's no hook event. The bug-equivalent failure mode for iOS Copilot is "instruction file's `applyTo:` glob doesn't match real paths" — which is much easier to debug (you can run `find . -path ...` against the glob).

## Verification

For Copilot, "did the rule fire?" is a different question than for Claude Code:

1. **`applyTo:` rule surfacing test** — open a file matching an instruction's `applyTo:` glob in Copilot Chat. Type *"@workspace what instruction files apply here?"* The matching instruction should be listed. If not, the glob doesn't match — fix it.
2. **Correction-capture test** — issue a correction in Chat, then invoke `/correction-capture`. The prompt should produce a draft instruction-file patch, not a "noted" acknowledgment.
3. **Build-gate test** — break a SwiftUI view, run a specialist task, confirm the specialist's `return:` block fails the build gate and the orchestrator rejects.
4. **`/commit-push-pr` test** — run on a trivial doc change with a `--dry-run` style abort instruction. Confirm the workflow is interruptible at each step.

If any of these fail in real-world use, treat as P0 — the Copilot enforcement layer is thinner than other harnesses, so the few mechanical pieces it has must work.

## Cross-links

- [`01-PRINCIPLES.md`](01-PRINCIPLES.md) § Principle 5 — what's runtime-enforced vs documented
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — `applyTo:` mechanics
- [`04-HANDOFF-SCHEMA.md`](04-HANDOFF-SCHEMA.md) — how the orchestrator's incoming-validation enforces the build-gate via Definition of Done
- [`08-IOS-COMMON-PITFALLS.md`](08-IOS-COMMON-PITFALLS.md) § Pitfall 19 — Copilot has no Stop event
- [`templates/prompts/correction-capture.prompt.md.template`](../templates/prompts/correction-capture.prompt.md.template) and [`commit-push-pr.prompt.md.template`](../templates/prompts/commit-push-pr.prompt.md.template)
