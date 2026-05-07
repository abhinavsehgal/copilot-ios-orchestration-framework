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
├── prompts/                                  ← Slash prompts (manually invoked: /<name>)
│   ├── correction-capture.prompt.md
│   ├── commit-push-pr.prompt.md
│   ├── verify-build.prompt.md
│   └── <YOUR-WORKFLOW>.prompt.md             ← add as patterns recur
├── agents/                                   ← Custom agents (Chat dropdown / @-mention)
│   ├── <your-app>-orchestrator.md
│   ├── ios-ui.md
│   ├── ios-data.md
│   ├── ios-network.md
│   ├── ios-tests.md
│   ├── ios-release.md
│   ├── ios-privacy.md                        ← REVIEW-ONLY (no Edit/Write tools)
│   ├── ios-perf.md
│   └── ios-bg.md
└── chatmodes/                                ← Optional personas (Chat dropdown)
    └── ios-architect.chatmode.md             ← rare; usually agents are enough

docs/                                          ← Project's own documentation
├── ai-context/                                ← Tier 1: orientation maps for Copilot
│   ├── INDEX.md                               ← The router agents read first
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
└── _archive/<YYYY-MM>/                        ← Tier 3: frozen history
    └── <archived-doc>.md
```

## How Copilot loads each file

### `.github/copilot-instructions.md` — auto-loaded everywhere
Loaded into the system prompt of **every** Copilot interaction across every surface (Chat, inline completions in Xcode + VS Code, code review on github.com, Cloud Agent). Keep this file thin (under 200 lines) — it's a router, not a manual.

What goes here:
- Project name, what it is in one sentence
- Golden rules (e.g. "always start on `develop`", "never push to `main`", "iOS 15.0 minimum")
- Mandatory workflow ("read INDEX.md before non-trivial tasks")
- Pointer to `docs/ai-context/INDEX.md`
- Routing tables (task type → which orientation map → which specialist agent)
- Branch + deploy table (develop → TestFlight, main → App Store)

What does NOT go here:
- Long architectural detail (goes in `docs/ARCHITECTURE.md`)
- The orchestrator's body prompt (goes in `.github/agents/<your-app>-orchestrator.md`)
- iOS-specific gotchas (go in `.github/instructions/<domain>.instructions.md`)

### `.github/instructions/<domain>.instructions.md` — auto-loaded by `applyTo:` match
Each file has YAML frontmatter declaring file paths it applies to:

```yaml
---
applyTo: "Sources/UI/**/*.swift,Sources/Views/**/*.swift,*.xib,*.storyboard"
---
```

When Copilot is editing a file matching the glob, the instruction body auto-loads into context. **No script. No hook. No restart.** This is Copilot's platform-native rule-surfacing mechanism.

Body length cap: ~3,000 chars for code-review-relevant rules (Copilot code review reads only the first ~4,000 chars of each instruction file). Longer instruction files OK for non-review contexts but front-load the most important rules.

### `.github/prompts/<name>.prompt.md` — manually invoked via `/<name>`
Invoked by typing `/<name>` in Copilot Chat. The prompt body becomes the chat input. Pre-existing slash inputs:

- `/commit-push-pr` — daily commit→PR loop with iOS golden rules baked in (xcodebuild gate, no-main-pushes, no `--force`, no committing `.cer`/`.mobileprovision`)
- `/correction-capture` — when you correct Copilot, this prompt walks it through hardening the lesson into an instruction file
- `/verify-build` — runs `xcodebuild` and surfaces failure tail to context

You add more as patterns recur in your project.

### `.github/agents/<NAME>.md` — selectable from Chat dropdown OR auto-routed
Each agent has frontmatter declaring its scope, tool allowlist, and behavioral rules:

```yaml
---
name: ios-data
description: CoreData / SwiftData / Realm / file storage specialist
tools: Read, Edit, MultiEdit, Bash, Grep, Glob, mcp__xcode__*
target: vscode, github-copilot
disable-model-invocation: false
---
```

The orchestrator agent has tools restricted to `Read, Grep, Glob, Bash` (no Edit/Write — it can only orchestrate, not implement). REVIEW-ONLY agents like `ios-privacy` have only `Read, Grep, Glob`. Implementation specialists like `ios-ui` have full Read+Edit+Bash.

### `.github/chatmodes/<NAME>.chatmode.md` — optional persona
Chat modes are simpler than agents — pure system-prompt overlays without tool restrictions or specific routing. Use sparingly. Default to agents for anything that's a real role.

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

## Cross-links

- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — what each specialist owns
- [`05-INSTRUCTIONS-AND-PROMPTS.md`](05-INSTRUCTIONS-AND-PROMPTS.md) — `applyTo:` mechanics and prompt-file syntax
- [`07-FOLDER-STRUCTURE.md`](07-FOLDER-STRUCTURE.md) — three-tier docs detailed
- [`11-IOS-MCP-CATALOG.md`](11-IOS-MCP-CATALOG.md) — MCP servers worth wiring up
