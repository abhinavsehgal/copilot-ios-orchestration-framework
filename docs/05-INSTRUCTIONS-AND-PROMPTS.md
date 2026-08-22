# 05 — Instructions, Skills and Prompts (iOS-flavored)

Three patterns for capturing knowledge that doesn't belong in agents: **instruction files** (path-globbed invariants), **agent skills** (repeatable workflows that work on every Copilot surface), and **prompt files** (IDE-only repeatable prompts invoked via slash command) — and how to use each for an iOS project.

> v1.0 of this chapter treated prompt files as the home for repeatable workflows. That was wrong: Agent Skills are a documented, cross-surface primitive and are the Copilot equivalent of Claude Code skills; prompt files are an IDE convenience. Corrected in v1.1.0 (verified 2026-08-22).

## Path-globbed instructions

Each `.github/instructions/<NAME>.instructions.md` file has YAML frontmatter declaring which paths it applies to. When Copilot is editing any matching file, the instruction body auto-loads into context.

### Anatomy of an instruction file

```yaml
---
applyTo: "Sources/**/Views/**/*.swift,Sources/Views/**/*.swift,*.xib,*.storyboard"
---

# SwiftUI / UIKit View Instructions

> **What this is.** Auto-loaded when editing any file matching `applyTo:` above.
>
> **Universal evidence rule.** Every claim about a View must cite path:line, an Apple HIG section, or a screenshot/snapshot test result. Don't claim "this view is accessible" without showing the relevant labels and traits.

## Hard rules

### 1. Avatars use `<img>` equivalents — never `next/image` ... wait, no, we're on iOS

(Joking — but a real example below.)

### 1. SwiftUI `@State` must be `private`

**Why:** `@State` is owned by the view; making it non-private breaks SwiftUI's identity invariants and causes silent re-render glitches.

**How to apply:** All `@State` declarations are `private var`. If you need to share state between views, use `@Binding` (downward) or `@StateObject` + `@ObservedObject` (upward).

**Cite when applying:** the offending line (e.g. `Sources/Views/ProfileView.swift:42`).

### 2. Don't capture `self` strongly in `Task { }` inside `@MainActor` views

**Why:** A strong-captured `self` in a long-running Task creates a retain cycle with the view's `@StateObject`. The view never deallocates.

**How to apply:** Use `[weak self]` or extract the work into a free function. If the work is genuinely view-scoped, use `.task { }` modifier (auto-cancels on view-disappear).

## What NOT to do

- Don't use `UIWebView` (deprecated since iOS 12; App Store rejection risk)
- Don't use `UIAlertView` / `UIActionSheet` (deprecated; use `UIAlertController`)
- Don't mix `setNeedsLayout` with SwiftUI layout passes
- Don't subclass `UIView` for visual effects when `Layer` masks would do (perf)

## Cross-links

- `docs/ai-context/swiftui-patterns.md` — orientation map for SwiftUI architecture
- `.github/instructions/concurrency.instructions.md` — Task / async-await rules
- `.github/instructions/info-plist.instructions.md` — required usage descriptions
```

### Glob examples for an iOS workspace

```yaml
# UI files (SwiftUI + UIKit + Interface Builder)
applyTo: "Sources/Views/**/*.swift,Sources/**/*View.swift,*.xib,*.storyboard"

# Networking layer
applyTo: "Sources/Networking/**/*.swift,Sources/**/*Client.swift,Sources/**/Endpoint*.swift"

# CoreData (model + extensions)
applyTo: "**/*.xcdatamodeld/**,Sources/Persistence/**/*.swift,Sources/**/Model+*.swift"

# Tests
applyTo: "Tests/**/*.swift,**/*Tests.swift,**/*UITests.swift"

# Info.plist + entitlements + xcconfig
applyTo: "**/Info.plist,**/*.entitlements,**/*.xcconfig"

# Fastlane
applyTo: "fastlane/**/*,Fastfile,Appfile,Matchfile"

# Multi-platform target subset
applyTo: "Sources/iOS/**/*.swift,Sources/Shared/**/*.swift"

# Pod / SPM manifests
applyTo: "Package.swift,Podfile,Podfile.lock,Cartfile"
```

### Size budget and front-loading for code review

The current docs give two limits (verified 2026-08-22): the code-review tutorial says to **limit any single instruction file to about 1,000 lines**, and the repository-instructions page says **instructions must be no longer than 2 pages**. The earlier "code review reads only the first ~4,000 chars" sentence is no longer on the docs and is not a cap you should design around (Pitfall 5). Still front-load the rules most likely to apply during PR review — the reader under the tightest budget sees the top of the file first:

```markdown
---
applyTo: "Sources/Views/**/*.swift"
---

# SwiftUI View Instructions

[the code-review-relevant Hard rules first]

## Reference (read fully when implementing)

[Longer rationale, examples, full pattern explanations]
```

Front-loading is good practice, not a hard limit. To keep an instruction out of code review entirely, add `excludeAgent: "code-review"` to its frontmatter.

### Length guidance

Keep each instruction file ≤ 150 lines — well inside the documented ~2-page budget. If you exceed, split:
- `swiftui-views.instructions.md` (rendering rules)
- `swiftui-state.instructions.md` (state management rules)
- `swiftui-accessibility.instructions.md` (a11y rules)

with three different `applyTo:` globs that may overlap in interesting ways.

## Agent skills (`.github/skills/<name>/SKILL.md`)

Repeatable multi-step workflows that load on **every** Copilot surface. This is the Copilot equivalent of Claude Code skills, and the home for anything the cloud agent or the CLI must be able to run — an `xcodebuild` verification, the commit→PR loop, the engineering playbook.

### Location

- Project: `.github/skills/<skill-name>/SKILL.md` (one directory per skill). Copilot also discovers `.claude/skills/` and `.agents/skills/`.
- Personal: `~/.copilot/skills/`.

### Frontmatter

| Field | Required | Notes |
|---|---|---|
| `name` | yes | Lowercase with hyphens; matches the directory name. |
| `description` | yes | One sentence — when to use this skill. Drives automatic loading by relevance. |
| `license` | no | |
| `allowed-tools` | no | Tools the skill may use. |
| `argument-hint` | no | VS Code only. |
| `user-invocable` | no | VS Code only — whether `/skill-name` is offered to the user. |
| `disable-model-invocation` | no | VS Code only — prevents automatic loading. |

### Invocation

Type `/<skill-name>` in chat, or let Copilot load the skill automatically when the task matches its `description`.

### Surfaces

Supported by the cloud agent, code review, the Copilot CLI, the Copilot app, and agent mode in VS Code and JetBrains (verified 2026-08-22). **Prompt files are IDE-only** — VS Code's own guidance for a prompt that agents on the Agent Host don't pick up is "convert it to an agent skill". If a workflow must run from an issue assignment or from `copilot -p` in CI, it is a skill.

### Pre-built skills in this framework

| Skill | What it does |
|---|---|
| `/<your-app>-engineering` | The six-gate playbook + evidence ladder ([`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md)); start here when no more specific workflow fits |
| `/commit-push-pr` | iOS-pre-filled commit→PR loop (xcodebuild gate, no main pushes, no `--force`, no committing `.cer`/`.mobileprovision`/`.p12`) |
| `/verify-build` | Runs `xcodebuild -workspace … -scheme … build`, surfaces the failure tail, reports a killed build as inconclusive |
| `/correction-capture` | When you correct Copilot, walks it through hardening the lesson into an instruction-file patch (the `agentStop` hook in Chapter 10 triggers the same discipline automatically once installed) |

Skill structure mirrors the prompt anatomy below — frontmatter, "when to invoke", workflow steps, Definition of Done — with `name:` added. See `templates/skills/`.

## Prompt files (`.github/prompts/<name>.prompt.md`) — IDE-only

Manually invoked by typing `/<name>` in Copilot Chat in VS Code / Visual Studio / JetBrains. The body becomes the chat input. **Not loaded by the cloud agent or the Copilot CLI.** Write one when a workflow is only ever started by hand in the IDE and needs `${input:…}` prompting; prefer a skill when it must run on the cloud agent or the CLI, or should load automatically. The v1.0 `/correction-capture`, `/commit-push-pr` and `/verify-build` prompt-file templates remain in `templates/prompts/` for IDE use; the skills above supersede them.

### Prompt anatomy

```yaml
---
description: <ONE_SENTENCE_DESCRIPTION — appears in /command picker>
---

# /<command-name>

> **Invocation.** Type `/<command-name>` in Copilot Chat.

## When to invoke

- <BULLET_LIST>

## When NOT to invoke

- <BULLET_LIST_OF_ANTI_TRIGGERS>

## Inputs

- `${input:goal:What is the goal?}`
- `${input:scheme:Which Xcode scheme should I build?}`

## Workflow

### 1. <Step name>
<2-4 sentences>

### 2. <Step name>
<...>

## Definition of Done

1. <ARTIFACT_1>
2. <ARTIFACT_2>
3. **Instruction files that applied** (the auto-loaded `.github/instructions/*.instructions.md` paths during this work)
```

### Example: a custom `/profile-view-tweak` prompt file (IDE-only; make it a skill if the cloud agent or CLI must run it)

If your team frequently tweaks the profile screen's accessibility, an iOS-specific prompt could be:

```yaml
---
description: Adjust ProfileView accessibility — labels, traits, dynamic type, dark mode contrast.
---

# /profile-view-tweak

## Inputs
- ${input:change:What accessibility property to adjust?}

## Workflow
1. Read `Sources/Views/ProfileView.swift` — note the current `.accessibilityLabel` / `.accessibilityHint` / `.accessibilityTraits` calls
2. Read `docs/ai-context/swiftui-patterns.md` § Accessibility
3. Apply the change with the project's standard pattern
4. Run snapshot tests: `xcodebuild test -only-testing:ProfileViewTests`
5. Verify on a Dynamic Type XL preview (file + line citation)
6. Report files changed + tests run + accessibility audit results
```

## Instructions vs skills vs prompt files vs agents — when to use which

| If the knowledge is... | Put it in... |
|---|---|
| A path-scoped invariant ("don't do X when editing Y") | Instruction file |
| A multi-step workflow that recurs, on any surface (cloud agent, CLI, IDE) | Agent skill |
| A multi-step prompt only ever started by hand in the IDE, with `${input:…}` | Prompt file |
| A persona with its own scope and Definition of Done | Custom agent |
| What is true right now / what we learned / what we deferred | `PROJECT.md` / `LEARNINGS.md` / `docs/<AREA>_BACKLOG.md` (Chapter 12) |

When something feels like it could be two of those: prefer instruction file > skill > prompt file > agent. A skill and a prompt file with the same name are confusing — keep one.

## Cross-links

- [`02-ARCHITECTURE.md`](02-ARCHITECTURE.md) — where these files live in your repo
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — which specialist auto-receives which instruction file
- [`10-MECHANICAL-ENFORCEMENT.md`](10-MECHANICAL-ENFORCEMENT.md) — the hooks that automate correction-capture and the build gate
- The `templates/instructions/` directory — six pre-built iOS instruction templates
- The `templates/skills/` directory — three pre-built iOS skills (+ `templates/engineering-playbook-skill.md.template`)
- The `templates/prompts/` directory — the v1.0 IDE-only prompt-file forms
