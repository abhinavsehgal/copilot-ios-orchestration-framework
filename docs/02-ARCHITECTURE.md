# 02 — Architecture

The framework lives entirely inside two folders in your iOS repo:

```
.github/                                      ← Copilot's customization surface
├── copilot-instructions.md                   ← Repository-wide router (auto-loaded on EVERY interaction)
├── instructions/                             ← Path-globbed instruction files (auto-loaded by applyTo: match)
│   ├── swiftui.instructions.md
│   ├── concurrency.instructions.md
│   ├── coredata.instructions.md
│   ├── networking.instructions.md
│   ├── info-plist.instructions.md
│   ├── code-signing.instructions.md
│   └── <YOUR-DOMAIN>.instructions.md         ← add as your project grows
├── skills/                                   ← Agent Skills — repeatable workflows, CROSS-SURFACE (v1.1)
│   ├── <your-app>-engineering/SKILL.md       ← six-gate playbook + evidence ladder (/<your-app>-engineering)
│   ├── commit-push-pr/SKILL.md               ← xcodebuild gate → commit → push → PR
│   ├── verify-build/SKILL.md                 ← xcodebuild + failure tail / "inconclusive"
│   ├── correction-capture/SKILL.md
│   └── <YOUR-WORKFLOW>/SKILL.md              ← add as patterns recur
├── prompts/                                  ← Prompt files — IDE-ONLY slash commands (/<name>)
│   └── <YOUR-IDE-ONLY-PROMPT>.prompt.md      ← the v1.0 correction-capture / commit-push-pr / verify-build prompts are superseded by the skills above
├── hooks/                                    ← Optional lifecycle hooks (Chapter 10) — COMMIT these
│   └── framework.json                        ← userPromptSubmitted / postToolUse / agentStop → .github/scripts/*.mjs
├── agents/                                   ← Custom agents (Chat dropdown / @-mention / VS Code subagent)
│   ├── <your-app>-orchestrator.agent.md      ← carries agents: [...] — the VS Code subagent allowlist
│   ├── ios-ui.agent.md
│   ├── ios-data.agent.md
│   ├── ios-network.agent.md
│   ├── ios-tests.agent.md
│   ├── ios-release.agent.md
│   ├── ios-privacy.agent.md                  ← REVIEW-ONLY (no Edit/Write tools)
│   ├── ios-perf.agent.md
│   └── ios-bg.agent.md
└── chatmodes/                                ← RETIRED — rename *.chatmode.md → agents/*.agent.md

docs/                                          ← Project's own documentation
├── ai-context/                                ← Tier 1: orientation maps for Copilot
│   ├── INDEX.md                               ← The router agents read first
│   ├── PROJECT.md                             ← Current truth: what is live on the App Store / TestFlight / backend (Chapter 12)
│   ├── LEARNINGS.md                           ← Decisions, failures, corrections (Chapter 12)
│   ├── GLOSSARY.md                            ← One name per concept (Chapter 12)
│   ├── HANDOFF_SCHEMA.md                      ← Bidirectional handoff contract
│   ├── ORCHESTRATION_SPOONFEEDER.md           ← Human-readable usage guide for the team
│   ├── swiftui-patterns.md
│   ├── coredata-schema.md
│   ├── networking-architecture.md
│   ├── release-process.md
│   └── <AREA>.md                              ← add per project area
├── ARCHITECTURE.md                            ← Tier 2: canonical refs (humans + agents)
├── BUILD_SYSTEM.md
├── PRIVACY_AND_DATA.md
├── DEPENDENCIES.md
├── CHANGELOG.md
├── <AREA>_BACKLOG.md                          ← Deferred work, one per area (Chapter 12)
└── _archive/<YYYY-MM>/                        ← Tier 3: frozen history
    └── <archived-doc>.md
```

## How Copilot loads each file

### `.github/copilot-instructions.md` — auto-loaded everywhere
Loaded into the system prompt of **every** Copilot interaction across every surface (Chat, inline completions in VS Code — and in Xcode where Copilot for Xcode reads it; not re-verified — code review on github.com, Cloud Agent). Keep this file thin (under 200 lines) — it's a router, not a manual.

What goes here:
- Project name, what it is in one sentence
- Golden rules (e.g. "always start on `develop`", "never push to `main`", "iOS 15.0 minimum")
- Mandatory workflow ("read INDEX.md before non-trivial tasks")
- Pointer to `docs/ai-context/INDEX.md`
- Routing tables (task type → which orientation map → which specialist agent)
- Branch + deploy table (develop → TestFlight, main → App Store)

What does NOT go here:
- Long architectural detail (goes in `docs/ARCHITECTURE.md`)
- The orchestrator's body prompt (goes in `.github/agents/<your-app>-orchestrator.agent.md`)
- iOS-specific gotchas (go in `.github/instructions/<domain>.instructions.md`)

### `.github/instructions/<domain>.instructions.md` — auto-loaded by `applyTo:` match
Each file has YAML frontmatter declaring file paths it applies to:

```yaml
---
applyTo: "Sources/UI/**/*.swift,Sources/Views/**/*.swift,*.xib,*.storyboard"
---
```

When Copilot is editing a file matching the glob, the instruction body auto-loads into context. **No script. No hook. No restart.** This is Copilot's platform-native rule-surfacing mechanism — the one pattern that never needed a hook, even now that Copilot has them (Chapter 10).

Size budget: the current docs say any single instruction file should stay under about 1,000 lines and repository instructions "no longer than 2 pages" (verified 2026-08-22). The older "code review reads only the first ~4,000 chars" sentence is no longer on the docs — front-load the most important rules as good practice, not because of a cap (Pitfall 5).

### `.github/skills/<name>/SKILL.md` — invoked as `/<name>` on EVERY surface
Agent Skills are the **cross-surface** primitive for repeatable workflows — the Copilot equivalent of Claude Code skills. One directory per skill, `SKILL.md` inside, frontmatter `name` (= the directory name) + `description`. Invoked as `/<name>` or auto-loaded when the task matches the description. Supported by the cloud agent, code review, the Copilot CLI, VS Code and JetBrains (verified 2026-08-22). The framework ships:

- `/<your-app>-engineering` — the six-gate playbook + evidence ladder (Chapter 12)
- `/commit-push-pr` — daily commit→PR loop with iOS golden rules baked in (xcodebuild gate, no-main-pushes, no `--force`, no committing `.cer`/`.mobileprovision`)
- `/verify-build` — runs `xcodebuild`, surfaces the failure tail, reports a killed build as inconclusive
- `/correction-capture` — when you correct Copilot, walks it through hardening the lesson into an instruction file

You add more as patterns recur in your project. See Chapter 5 for the full frontmatter.

### `.github/prompts/<name>.prompt.md` — IDE-only slash commands
Invoked by typing `/<name>` in Copilot Chat in VS Code / Visual Studio / JetBrains. The prompt body becomes the chat input. **Not loaded by the cloud agent or the Copilot CLI** — VS Code's own advice for a prompt an agent must run is "convert it to an agent skill". v1.0 shipped `/commit-push-pr`, `/verify-build` and `/correction-capture` as prompt files; v1.1 ships them as skills (above) and keeps the prompt-file templates for IDE-only use.

### `.github/hooks/<name>.json` — optional lifecycle hooks (Chapter 10)
Declares commands to run on `sessionStart`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop` and other events. The cloud agent loads only `.github/hooks/*.json`; the CLI also reads `~/.copilot/hooks/`; VS Code also reads `.claude/settings.json`. This is where the `xcodebuild` build-gate, the doc-freshness gate and correction-capture live once you decide you need them — see [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) and `templates/hooks/`.

### `.github/agents/<NAME>.agent.md` — selectable from Chat dropdown, @-mentioned, or invoked as a subagent
File name `<name>.agent.md` (a bare `.md` still loads; `.agent.md` is the current convention). Each agent has frontmatter declaring its scope, tool allowlist, and behavioral rules:

```yaml
---
name: ios-data
description: CoreData / SwiftData / Realm / file storage specialist
tools: read, edit, bash, grep, glob, xcode/*
target: vscode, github-copilot
disable-model-invocation: false
---
```

The orchestrator additionally carries `agents: [ios-ui, ios-data, …]` — in VS Code that list is a real allowlist of which agents it may invoke as subagents (`*` = all; never on an orchestrator); the cloud agent ignores the field and reads the routing table in the body. Specialists carry no `agents:` — they return `recommended_next_agent` rather than chaining (Chapter 3).

The orchestrator agent has tools restricted to `read, search, grep, glob, bash` (no `edit` — it can only orchestrate, not implement). REVIEW-ONLY agents like `ios-privacy` have only `read, search, grep, glob` (no shell either — a shell can write). Implementation specialists like `ios-ui` add `edit` and `bash`. These are the tool aliases on the GitHub custom-agents reference (verified 2026-08-22); VS Code recognises its own identifiers (`search/codebase`, `edit`, `runCommands`, …) and silently IGNORES an unknown name, so verify the list in your IDE's agent configuration UI.

### `.github/chatmodes/<NAME>.chatmode.md` — RETIRED
Custom chat modes were the earlier name for custom agents and are retired (verified 2026-08-22). Rename any existing file to `.github/agents/<name>.agent.md`; `description`, `tools` and `model` carry over unchanged. Do not create new chat-mode files.

## Variations

### Tuist or Bazel workspace
The folder layout above assumes a standard Xcode workspace (`MyApp.xcworkspace` + per-target groups). If you use Tuist (project generation) or Bazel (BUILD files everywhere), the `applyTo:` globs change but the architecture doesn't:

```yaml
# Tuist
applyTo: "Tuist/**/*.swift,Targets/**/Sources/**/*.swift"

# Bazel
applyTo: "ios/**/Sources/**/*.swift,ios/**/BUILD.bazel"
```

### Mixed Swift + Objective-C
List both extensions in `applyTo:` for instruction files that span both:

```yaml
applyTo: "Sources/Networking/**/*.swift,Sources/Networking/**/*.{h,m}"
```

### Multi-platform (iOS + watchOS + visionOS in one workspace)
Add per-platform specialists (`ios-watch`, `ios-vision`) and per-platform instruction files (`watchos-complications.instructions.md`). The orchestrator routes based on the file paths in the diff.

### Pure iOS framework / SDK (not an app)
Drop `ios-release` (no App Store), drop `ios-bg` (no app lifecycle), drop `ios-privacy` (consumers' problem). Add `ios-api-stability` (`@available` annotations, ABI stability, semver discipline) and `ios-docc` (DocC comment quality, public API surface).

### Multiple repositories (the app + its API repos, a shared Swift package, an Android sibling)
When the app and the backend it consumes live in separate repos, see **[`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md)** — the cloud agent cannot change more than one repository in a run, so cross-repo work is a workspace of per-repo installs plus shared contracts (an OpenAPI/GraphQL spec or a version-pinned SPM package), not one agent over everything. The iOS repo keeps the layout above unchanged.

## Cross-links

- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — what each specialist owns
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — `applyTo:` mechanics and prompt-file syntax
- [`07-FOLDER-STRUCTURE.md`](07-FOLDER-STRUCTURE.md) — three-tier docs detailed
- [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) — the hooks in `.github/hooks/`
- [`11-IOS-MCP-CATALOG.md`](11-IOS-MCP-CATALOG.md) — MCP servers worth wiring up
- [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md) — `PROJECT.md` / `LEARNINGS.md` / `GLOSSARY.md` / backlogs
- [`13-MULTI-REPO-WORKSPACES.md`](13-MULTI-REPO-WORKSPACES.md) — when the app is one repo among several
