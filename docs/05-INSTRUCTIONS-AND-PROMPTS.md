# 05 — Instructions and Prompts (iOS-flavored)

The two main customization mechanisms Copilot offers — and how to use both for an iOS project.

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

### Front-loading rules for code review

GitHub Copilot's code-review feature reads only the **first ~4,000 chars** of each instruction file. Front-load the rules most likely to apply during PR review:

```markdown
---
applyTo: "Sources/Views/**/*.swift"
---

# SwiftUI View Instructions

[~3,500 chars of code-review-relevant Hard rules]

## Reference (read fully when implementing)

[Longer rationale, examples, full pattern explanations]
```

Anything in the first 4k is seen by reviewer Copilot. Anything after is seen by implementer Copilot but skipped during review.

### Length cap

Keep each instruction file ≤ 150 lines / ~3,000 chars. If you exceed, split:
- `swiftui-views.instructions.md` (rendering rules)
- `swiftui-state.instructions.md` (state management rules)
- `swiftui-accessibility.instructions.md` (a11y rules)

with three different `applyTo:` globs that may overlap in interesting ways.

## Slash prompts (`.github/prompts/<name>.prompt.md`)

Manually invoked by typing `/<name>` in Copilot Chat. The body becomes the chat input. Use for repeatable workflows.

### Pre-built prompts in this framework

| Prompt | What it does |
|---|---|
| `/correction-capture` | When you correct Copilot, walks it through hardening the lesson into an instruction-file patch (since Copilot has no Stop hook to auto-do it) |
| `/commit-push-pr` | iOS-pre-filled commit→PR loop (xcodebuild gate, no main pushes, no `--force`, no committing `.cer`/`.mobileprovision`/`.p12`) |
| `/verify-build` | Runs `xcodebuild -workspace … -scheme … build` and surfaces failure tail |

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

### Example: a custom `/profile-view-tweak` slash prompt

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

## Cross-links

- [`02-ARCHITECTURE.md`](02-ARCHITECTURE.md) — where these files live in your repo
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — which specialist auto-receives which instruction file
- The `templates/instructions/` directory — six pre-built iOS instruction templates
- The `templates/prompts/` directory — three pre-built iOS slash-prompt templates
