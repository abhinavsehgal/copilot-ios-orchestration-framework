# Copilot iOS Orchestration Framework

> **Version 1.0.0** ([changelog](CHANGELOG.md)) · MIT license · iOS-only by design
>
> **Purpose.** A reusable multi-agent orchestration setup for [GitHub Copilot](https://docs.github.com/en/copilot) tailored specifically for iOS engineering teams. Drops into any existing iOS codebase (UIKit / SwiftUI / Combine / async-await / Objective-C bridges) in 2-4 hours. Includes iOS-specialist agents, iOS-flavored instruction templates, an iOS pitfalls catalog, and a curated MCP integration list for the iOS toolchain.

This is **a sibling to**, not a replacement for, the stack-agnostic [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework). If your project is web, backend, or polyglot, use that one. If you ship to the App Store, this one is built for you.

---

## Why an iOS-only framework

The general Copilot framework is stack-agnostic by design — but its examples, specialist names, and bootstrap prompts lean web (Next.js, REST APIs, Stripe). An iOS adopter has to mentally translate `frontend-ui` → `ios-ui`, `npm run build` → `xcodebuild`, `next/image` rules → SwiftUI rendering rules, and so on. That works, but it leaks effort.

This framework eliminates that translation. Out of the box:

- **iOS specialist roster** — `ios-ui`, `ios-data`, `ios-network`, `ios-tests`, `ios-release`, `ios-privacy` (REVIEW-ONLY), `ios-perf`, `ios-bg`. Each one has a specific scope, tool allowlist, and Definition of Done.
- **iOS placeholders pre-filled** — `xcodebuild` build commands, Swift / Objective-C globs, `Info.plist` paths, `*.xcconfig` rules, code-signing scopes, App Store Connect references.
- **iOS pitfalls catalog** — 20+ recurring iOS gotchas (background `viewContext.save()`, `@MainActor` capture leaks, Info.plist usage descriptions, code signing rotation, etc.).
- **MCP integration catalog** — the curated list of iOS-relevant MCP servers (simulator, Xcode, mobile UI automation, App Store Connect, Sentry, GitHub) with what each is for and when to install.
- **iOS-anchored bootstrap prompts** — Copilot is told upfront *"this is an iOS app — propose iOS-shaped specialists"* so it doesn't waste a turn proposing `frontend-ui`.

---

## Who this is for

- **iOS engineering teams** using Copilot on production iOS codebases (App Store apps, enterprise apps, framework SDKs)
- Projects with **mixed Swift + Objective-C**, **multiple modules** (Pods or SPM), **CI on Xcode Cloud / GitHub Actions / Bitrise**, or **App Store privacy + compliance** concerns
- Anyone whose Copilot has hallucinated a SwiftUI lifecycle, suggested wrong concurrency patterns, or proposed `@objc` glue that breaks ABI compatibility

If your iOS project is a solo prototype or under ~5,000 lines, default Copilot is enough. Adopt this when complexity passes the threshold.

---

## What's in the box

```
copilot-ios-orchestration-framework/
├── README.md                              ← this file
├── CHANGELOG.md
├── LICENSE
│
├── docs/                                  ← framework documentation (11 chapters)
│   ├── 01-PRINCIPLES.md                   ← seven core principles
│   ├── 02-ARCHITECTURE.md                 ← .github/ + iOS-flavored docs/ai-context/ layout
│   ├── 03-IOS-SPECIALISTS-GUIDE.md        ← the iOS specialist roster + scope tables
│   ├── 04-HANDOFF-SCHEMA.md               ← bidirectional schema with iOS examples
│   ├── 05-INSTRUCTIONS-AND-PROMPTS.md     ← path-globbed instructions for iOS file paths
│   ├── 06-INVOCATION-MODES.md             ← Chat / Edit / Cloud Agent / CLI on iOS workflow
│   ├── 07-FOLDER-STRUCTURE.md             ← three-tier docs organization
│   ├── 08-IOS-COMMON-PITFALLS.md          ← 20+ iOS-specific lessons
│   ├── 09-IOS-RUNBOOK.md                  ← step-by-step iOS bootstrap (~2-4 hours)
│   ├── 10-MECHANICAL-ENFORCEMENT.md       ← Copilot has no hooks; what to use instead
│   └── 11-IOS-MCP-CATALOG.md              ← curated iOS MCP servers + when to install each
│
├── prompts/                               ← ready-to-paste Chat prompts
│   ├── INVENTORY-PROMPT.md                ← read-only iOS-anchored project scan (run first)
│   ├── BOOTSTRAP-PROMPT.md                ← generate all framework files for your iOS project
│   └── REFINEMENT-PROMPT.md               ← post-bootstrap audit (run after 2-3 weeks)
│
└── templates/                             ← iOS-specific drop-in templates
    ├── copilot-instructions.md.template            ← .github/copilot-instructions.md (root router)
    ├── HANDOFF_SCHEMA.md.template
    ├── INDEX.md.template
    ├── SPOONFEEDER.md.template
    ├── archive-README.md.template
    │
    ├── agents/
    │   ├── orchestrator-agent.md.template
    │   ├── ios-ui-agent.md.template               ← SwiftUI / UIKit / accessibility
    │   ├── ios-data-agent.md.template             ← CoreData / SwiftData / Realm / file storage
    │   ├── ios-network-agent.md.template          ← URLSession / async-await / Combine / certs
    │   ├── ios-tests-agent.md.template            ← XCTest / XCUITest / snapshot tests
    │   ├── ios-release-agent.md.template          ← Fastlane / TestFlight / App Store Connect
    │   ├── ios-privacy-agent.md.template          ← REVIEW-ONLY: Info.plist / ATT / nutrition labels
    │   ├── ios-perf-agent.md.template             ← Instruments / hangs / memory / launch time
    │   ├── ios-bg-agent.md.template               ← BGTaskScheduler / silent push / lifecycle
    │   ├── review-only-agent.md.template          ← generic REVIEW-ONLY shape (for legal-compliance etc.)
    │   └── specialist-agent.md.template           ← generic specialist shape (for adding more)
    │
    ├── instructions/
    │   ├── swiftui.instructions.md.template
    │   ├── concurrency.instructions.md.template
    │   ├── coredata.instructions.md.template
    │   ├── networking.instructions.md.template
    │   ├── info-plist.instructions.md.template
    │   └── code-signing.instructions.md.template
    │
    └── prompts/
        ├── prompt.md.template                     ← generic prompt-file shape
        ├── chatmode.md.template                   ← optional persona shape
        ├── correction-capture.prompt.md.template  ← /correction-capture (iOS-flavored)
        ├── commit-push-pr.prompt.md.template      ← /commit-push-pr (xcodebuild pre-filled)
        └── verify-build.prompt.md.template        ← /verify-build (xcodebuild pre-filled)
```

---

## Quick start

### Scenario A — Brownfield iOS project (most common)

```bash
# 1. Verify Copilot is signed in inside Xcode (or VS Code with the Swift extension if you prefer)
#    https://docs.github.com/en/copilot/getting-started

# 2. Open your iOS project root in your IDE
cd ~/path/to/MyiOSApp

# 3. Make sure git is clean — bootstrap creates new files on a fresh branch
git status
git checkout -b setup/copilot-ios-framework

# 4. Run the iOS inventory pass (read-only — proposes what to build)
#    Open this repo's prompts/INVENTORY-PROMPT.md
#    Copy contents → paste into Copilot Chat
#    Replace <framework path> with the absolute path to this repo on your machine

# 5. Review/adjust the proposed iOS specialist list + answer Open Questions

# 6. Run the iOS bootstrap pass (creates all .github/ + docs/ai-context/ files)
#    Paste prompts/BOOTSTRAP-PROMPT.md (in the same Chat session)

# 7. Verify locally
ls .github/agents/                             # should list ios-ui, ios-data, etc.
ls .github/instructions/                       # should list domain-specific instruction files
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build  # should still succeed

# 8. Try a real task via the orchestrator
#    Type @<your-app>-orchestrator in Copilot Chat with a small bug or feature
#    Verify the handoff schema works end-to-end

# 9. Commit, PR, merge
git add .github/ docs/ai-context/ docs/_archive/
git commit -m "chore: bootstrap Copilot iOS orchestration framework"
git push -u origin setup/copilot-ios-framework
gh pr create --base develop --title "Bootstrap Copilot iOS orchestration"
```

Estimated time: **2-4 hours** (split across phases — see [`docs/09-IOS-RUNBOOK.md`](docs/09-IOS-RUNBOOK.md)).

### Scenario B — New iOS project (greenfield)

Skip INVENTORY-PROMPT (no existing config to scan). Paste BOOTSTRAP-PROMPT directly with your stack details. Estimated time: **45 min – 1 hour**.

### Scenario C — Just want to read the framework

Read in this order: README → [01-PRINCIPLES](docs/01-PRINCIPLES.md) → [03-IOS-SPECIALISTS-GUIDE](docs/03-IOS-SPECIALISTS-GUIDE.md) → [09-IOS-RUNBOOK](docs/09-IOS-RUNBOOK.md). ~45 minutes.

---

## What this framework is opinionated about

- **iOS scope is the project's own definition of "iOS."** That includes universal apps (iOS + iPadOS), Mac Catalyst, watchOS / tvOS / visionOS companions if part of the same workspace, and iOS framework SDKs. Pure macOS / pure server-side Swift projects should use the [stack-agnostic framework](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) instead.
- **Universal evidence rule** — every claim Copilot makes (about a CoreData entity, a runtime version, an entitlement, an `Info.plist` key, an App Store guideline) must cite a file:line, schema, or rule. No vibes-based claims pass the orchestrator's validation.
- **REVIEW-ONLY for privacy + App Store compliance.** The `ios-privacy` agent has zero Write/Edit tools — it can read `Info.plist`, entitlements files, and privacy nutrition labels, but it cannot change them. That's a runtime lock, not a sticky note. (See [Principle 5](docs/01-PRINCIPLES.md#5-tools-are-runtime-enforced-everything-else-is-documentation-discipline).)
- **MCPs are recommendations, not requirements.** [`docs/11-IOS-MCP-CATALOG.md`](docs/11-IOS-MCP-CATALOG.md) lists what to consider; you install what you need. The framework works with zero MCPs installed (using xcodebuild directly through Bash) and gets sharper with each MCP added.

## What this framework does NOT include

- **iOS code samples or sample app.** Templates have placeholders. You fill in for your project.
- **Vendor lock-in.** No App Store accounts, no third-party SaaS dependencies. Only the documented Copilot customization surface (`.github/instructions/`, `.github/prompts/`, `.github/agents/`, `.github/chatmodes/`, `.github/copilot-instructions.md`) plus optional MCP integrations.
- **Runtime hooks.** Copilot doesn't have programmable lifecycle hooks. Mechanical enforcement happens via `applyTo:` auto-loading + IDE auto-fix + pre-commit. See [`docs/10-MECHANICAL-ENFORCEMENT.md`](docs/10-MECHANICAL-ENFORCEMENT.md) and Pitfall 19 in the [common pitfalls doc](docs/08-IOS-COMMON-PITFALLS.md).

---

## Customization for your project

Three things change per project:

1. **Project name + slug** (used to name the orchestrator: `<your-app>-orchestrator`)
2. **Specialist scope** (which of the 8 default iOS specialists you keep, which you drop, which you add — e.g. add `ios-watch` for watchOS-heavy apps, drop `ios-bg` if your app has no background work)
3. **Path globs in instructions** (matched to your actual module / file layout — Pods vs SPM vs mixed)

Everything else (the handoff schema, evidence rule, failure_condition pattern, three-tier docs, invocation modes) stays the same across projects.

See [`docs/03-IOS-SPECIALISTS-GUIDE.md`](docs/03-IOS-SPECIALISTS-GUIDE.md) for the specialist roster and [`docs/05-INSTRUCTIONS-AND-PROMPTS.md`](docs/05-INSTRUCTIONS-AND-PROMPTS.md) for instruction-file authoring.

## Maintenance

After bootstrapping, the framework needs minimal upkeep:

- **Per task:** instruction files accumulate naturally as production teaches you new gotchas (1-2 per month is normal — code signing rotation, new entitlement keys, OS-version-specific bugs)
- **Per quarter:** run [`prompts/REFINEMENT-PROMPT.md`](prompts/REFINEMENT-PROMPT.md) to audit specialist scope drift, archive stale docs, decide if MCPs should be added/removed
- **Per major iOS release:** re-read [`docs/08-IOS-COMMON-PITFALLS.md`](docs/08-IOS-COMMON-PITFALLS.md) and add new pitfalls (every iOS major brings 3-5 new gotchas — privacy manifests, predictable IDs, etc.)

---

## Frequently asked

**Q: Why a separate framework instead of just adding iOS examples to the parent?**
The parent framework's strength is being stack-agnostic. Adding heavy iOS-specific content there would make it noisier for web / backend adopters. iOS engineers benefit from a focused framework that "just knows" the toolchain.

**Q: Can I use this on a mixed iOS + Android codebase (React Native / Flutter / Kotlin Multiplatform)?**
Partially. The iOS-specific parts work for the iOS target. For Android, fork or pair with a sister framework (TBD). For React Native / Flutter shared logic, use the [stack-agnostic framework](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) and add iOS-only specialists from this one.

**Q: Does this work with Tuist / Bazel / Buck workspaces?**
Yes — the `applyTo:` globs adapt to whatever path layout your build system uses. The instruction examples assume a standard Xcode workspace, but the framework is build-system-agnostic.

**Q: Will the App Store reject my app because of this framework?**
No. The framework only ships markdown files in `.github/` and `docs/`. Nothing reaches your binary. App Store reviewers won't see any of this.

**Q: Is this Anthropic-official / GitHub-official?**
Neither. This is a community framework built on top of GitHub Copilot's documented customization surface. No special access required, no sponsorship.

**Q: Can I share with another iOS team?**
This repo is public, MIT licensed. Use freely.

## Companion frameworks

- [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) — the stack-agnostic parent. Use for non-iOS projects.
- [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) — the Claude Code equivalent. Use if your team uses Claude Code instead of (or alongside) Copilot.

## Contributing

iOS adopters' real-world experience is the main feedback channel for v1.x. If you discover an iOS-specific pitfall, a missing specialist, a better instruction-file pattern, or an MCP worth adding — open an issue or PR. The framework improves as it sees more projects.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.
