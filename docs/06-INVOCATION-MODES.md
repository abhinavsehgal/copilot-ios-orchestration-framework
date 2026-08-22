# 06 — Invocation Modes

How Copilot's different invocation surfaces interact with this framework, with iOS-specific nuances.

## The six surfaces Copilot offers

| Surface | Where | iOS use case |
|---|---|---|
| **Inline completions** | Inside any file, while typing | Auto-complete Swift code, fill in protocol conformances, suggest closures |
| **Inline Chat** (Cmd+I in VS Code, ⌃⌘I in Xcode) | Per-file context menu | Quick refactor or "explain this method" |
| **Chat panel** | VS Code sidebar / Xcode sidebar / GitHub.com / JetBrains tool window | Multi-turn work, agent selection, skills + slash prompts |
| **Code Review** | github.com PR | Auto-comment on PRs based on `applyTo:` instructions |
| **Cloud Agent** (autonomous coding agent) | github.com / Slack | Long-running autonomous tasks; runs in true isolation |
| **Copilot CLI, headless** (`copilot -p`) — v1.1 | Terminal / CI | Scripted runs of a skill or agent; cross-repo delegation (Chapter 13) |

## How each surface uses the framework

### Inline completions

- Loads `.github/copilot-instructions.md` (the thin router)
- Does NOT load `.github/instructions/<NAME>.instructions.md` automatically (those are Chat-context loads)
- Does NOT invoke agents

**Implication:** keep `.github/copilot-instructions.md` thin. Anything you put there gets loaded into every keystroke-level suggestion. Heavy stuff goes elsewhere.

For iOS specifically: don't put Swift coding standards in `.github/copilot-instructions.md` — put them in domain-scoped instruction files (`swiftui.instructions.md`, `concurrency.instructions.md`, etc.) so they only load in Chat for the relevant areas.

### Inline Chat (per-file)

- Loads `.github/copilot-instructions.md`
- Loads matching `.github/instructions/<NAME>.instructions.md` based on the active file
- Agent selection: usually inherits from the current Chat session; can be overridden

Best for: small refactors in a single file ("rename this property", "extract this view modifier", "add a docstring").

### Chat panel

- Loads `.github/copilot-instructions.md`
- Loads matching `.github/instructions/<NAME>.instructions.md` per file the chat references
- Agents available via dropdown OR `@<agent-name>`
- Skills available via `/<skill-name>` (or auto-loaded by relevance); IDE-only prompt files via `/<prompt-name>`

This is where the framework's value is highest. The orchestrator + specialist pattern is designed for this surface.

iOS-specific tip: when the active file is in `Sources/Views/`, the `swiftui.instructions.md` auto-loads. If you switch context to discuss a CoreData migration but stay in the same Chat, the `coredata.instructions.md` rules don't apply unless you reference a CoreData file. Use `@ios-data` to explicitly bring CoreData specialist context in regardless of active file.

### Code Review (github.com)

- Loads `.github/copilot-instructions.md`
- Loads matching `.github/instructions/*.instructions.md` based on changed files in the PR
- Loads agent skills (`.github/skills/`)
- Reads instruction files within the documented budget (about 1,000 lines per file / two pages — the old "first ~4,000 chars" cap is no longer on the docs, verified 2026-08-22) — still front-load review-relevant rules
- Does NOT run agents
- Does NOT run prompt files (IDE-only)

**Implication for iOS:** Code Review is good for catching `applyTo:`-matched rule violations on PRs (e.g. *"this PR uses `next/image` for an avatar"* — wait, wrong stack — *"this PR force-unwraps an optional in a public API; rule X says don't"*). It's not good for cross-domain analysis (use Chat or Cloud Agent for that).

### Cloud Agent (autonomous)

- Loads `.github/copilot-instructions.md`, matching instruction files, the assigned `.github/agents/*.agent.md`, skills from `.github/skills/`, and hooks from `.github/hooks/*.json` — and **only** from there (Chapter 10)
- Does NOT load prompt files (IDE-only — convert to a skill)
- Runs in **true isolation** — its context starts fresh; no leakage from any developer's local Chat
- **Per-repository:** "Copilot cannot make changes across multiple repositories in one run" — one branch, one PR per task (Chapter 13 for the app + its API repos)
- Long-running (multi-step builds, multi-file refactors)

iOS use cases:
- *"Migrate all `UIWebView` references to `WKWebView`"* (if any are still around — unlikely in 2026 but you get the idea)
- *"Add `@available` annotations to all public APIs in MyFrameworkSDK based on git blame"*
- *"Run `xcodebuild test`, file an issue for each failing test, link to the test file"*

The Cloud Agent's isolation is a real strength for iOS — when you ask it to refactor every CoreData context-handling site in the project, it doesn't have your local Chat's baggage. It re-reads the codebase, re-loads `coredata.instructions.md`, and produces a clean PR. Remember it has no signing material (Pitfall 20): a hook or skill that runs `xcodebuild` there must use `build`, never `archive`.

### Copilot CLI, headless (`copilot -p`) — the agentic CLI (v1.1)

The agentic `copilot` command in the terminal. Interactive by default; non-interactive with `copilot -p "<prompt>"`.

**Loads** (from the working directory): `.github/copilot-instructions.md`, instruction files, `AGENTS.md` / `CLAUDE.md`, custom agents, skills, and hooks (`.github/hooks/*.json` plus `~/.copilot/hooks/`). There is **no `--cwd` flag** — run it from the repo directory (or the worktree). `--add-dir=<path>` grants access to additional directories.

**Flags that matter for orchestration:**
- `--agent=<name>` — run as a specific custom agent (the orchestrator, for a scripted multi-domain task; `ios-tests` for a scheduled test triage).
- `--allow-tool=<tool>` / `--deny-tool=<tool>` / `--allow-all-tools` — tool permissions for a non-interactive run. A headless run that needs to edit or run `xcodebuild` must be granted that explicitly; `ios-privacy` should be run with nothing beyond read tools allowed. Never `--allow-all-tools` on a repo you don't fully trust — `-p` runs that repo's hooks without a prompt.
- `-s` / `--silent` — suppress non-output chatter for scripting.
- `--output-format` is **not documented** — parse the plain text output, or have the agent write a file.

**Use for:** CI jobs (`copilot -p "Run /verify-build" --agent=ios-tests --allow-tool=<run tools> -s` from the repo root), scheduled sweeps, and scripted delegation from a multi-repo workspace to this repo's orchestrator (Chapter 13). Because hooks load here, a `copilot -p` run is still governed by the `xcodebuild` build-gate / doc-freshness hooks the repository ships. (The older `gh copilot suggest` / `explain` extension has its own instruction system and runs no agents, skills or hooks — one-line shell questions only.)

(Custom chat modes are retired — rename `.chatmode.md` files to `.github/agents/<name>.agent.md`.)

## Surface-specific framework gotchas

### Inline completions don't enforce path-globbed instructions

If your `swiftui.instructions.md` says *"never use `@ObservedObject` in a top-level view; use `@StateObject`"* — Copilot Chat respects it (auto-loaded), but **inline completions while you're typing in a SwiftUI view file do not load the rule**. Inline completions only see the thin `.github/copilot-instructions.md`.

**Mitigation:** put the most-violated rules in `.github/copilot-instructions.md` itself with iOS prefix tags: *"iOS: SwiftUI top-level views use `@StateObject` not `@ObservedObject`."* The trade-off is a slightly heavier router, but inline completions get the benefit.

### Cloud Agent vs Chat: the `mcp-servers:` allowlist

Both honor the per-agent `mcp-servers:` allowlist. So if `ios-release` has `mcp-servers: app-store-connect, github`, the Cloud Agent running as `ios-release` can hit ASC; the same agent in your local Chat can hit ASC if you've installed those MCPs locally. Without the local install, the Cloud Agent has more capability than your IDE Copilot — that's expected.

### Code Review is read-only

Code Review can leave inline comments but cannot edit the PR. It cannot run `xcodebuild`. Don't put rules in your instruction files that require execution to verify. Front-load **observable** rules: *"this method has no @available annotation"* (observable from the diff) is fine; *"this code increases launch time by 100ms"* (requires Instruments) is not.

### Inline Chat is single-file context

Inline Chat does not see other files in the project. If you're refactoring a SwiftUI view that depends on a Networking layer change, inline Chat won't know about the Networking change. Use the Chat panel for cross-file work.

## When to use which surface

| Task | Best surface |
|---|---|
| Type the next line of Swift | Inline completions |
| "What does this method do?" | Inline Chat |
| "Refactor this view" (single file) | Inline Chat |
| "Add a streak feature" (multi-file) | Chat panel + orchestrator |
| "Catch obvious rule violations on this PR" | Code Review (auto) |
| "Run the test suite and triage failures" | Chat panel + `@ios-tests` + `/verify-build` |
| "Migrate every `UIWebView` to `WKWebView` across 80 files" | Cloud Agent |
| "Investigate this Crashlytics top-1 crash and propose a fix" | Cloud Agent + `@ios-perf` |
| "Every night, run the test plan and write a triage file" | `copilot -p … --agent=ios-tests` from the repo directory, in CI |
| "Add a field to the orders API and show it in the app" (two repos) | the workspace orchestrator → `copilot -p` into each repo (Chapter 13) |

## Surface support matrix (current)

| Feature | VS Code | JetBrains | Visual Studio | github.com Chat | Cloud Agent | Code review | CLI (`copilot`) |
|---|---|---|---|---|---|---|---|
| `.github/copilot-instructions.md` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `.github/instructions/*.instructions.md` (auto-load on path match) | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `AGENTS.md` (nearest wins) | ✅ root (nested behind `chat.useNestedAgentsMdFiles`) | not documented | not documented | not documented | ✅ | ✅ | ✅ |
| `.github/skills/*/SKILL.md` (agent skills) | ✅ | ✅ | not documented | not documented | ✅ | ✅ | ✅ |
| `.github/prompts/*.prompt.md` (slash commands, IDE-only) | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| `.github/agents/*.agent.md` (custom agents) | ✅ | ✅ | ✅ | ✅ | ✅ | — | ✅ (`--agent=`) |
| `agents:` subagent allowlist | ✅ | not documented | not documented | ❌ | ❌ (ignored) | — | not documented |
| `.github/hooks/*.json` (lifecycle hooks) | ✅ (also `.claude/settings.json`) | not documented | not documented | not documented | ✅ (`.github/hooks` only) | not documented | ✅ (also `~/.copilot/hooks/`) |
| `.github/chatmodes/*.chatmode.md` | RETIRED — rename to `.agent.md` | RETIRED | RETIRED | ❌ | ❌ | ❌ | ❌ |
| MCP servers per agent | ✅ | ✅ | ✅ | partial | partial | — | ✅ |

(Verified against docs.github.com and code.visualstudio.com on 2026-08-22. "not documented" means the current docs neither confirm nor deny it — do not assume either way. Surface support evolves rapidly; re-verify quarterly — Pitfall 23.) Xcode's Copilot integration is not in this matrix — it was not part of that verification pass; treat every row as unverified for Xcode and check in your own setup.

## Cross-links

- [`02-ARCHITECTURE.md`](02-ARCHITECTURE.md) — what files exist where, what each does
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — what auto-loads when; skills vs prompt files
- [`08-IOS-COMMON-PITFALLS.md`](08-IOS-COMMON-PITFALLS.md) § Pitfall 20 (Cloud Agent provisioning + signing), § Pitfall 23 (platform drift)
- [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) — which surfaces load hooks
- [`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md) — `copilot -p` as the cross-repo delegation mechanism
