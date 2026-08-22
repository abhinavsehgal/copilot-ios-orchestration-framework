# Copilot iOS Orchestration Framework

> **Version 1.1.0** ([changelog](CHANGELOG.md)) · MIT license · iOS-only by design
>
> **v1.1.0 (2026-08-22) — three months of production use on the stack-agnostic parent, folded into the iOS edition, and three platform claims retracted.** New chapters: **12 — Project truth, learnings and the evidence ladder** (`PROJECT.md` with a date-stamped "what is live on the App Store / TestFlight" table) and **13 — Multi-repo workspaces** (one iOS app repo + N API repos, with the `.code-workspace` + manifest + delegation pattern and version-pinned shared Swift packages). Eighteen new pitfalls — nine framework lessons and nine iOS-specific ones (APNs environment, `apns-topic`, the one-shot notification prompt, `WKWebView` route changes, Guideline 4.8, the `NSURLSession` cookie jar, orientation locks, replayed notification taps, build flavours). **Retracted, because the platform moved:** Copilot *does* have lifecycle hooks now (`.github/hooks/*.json` — chapter 10 is rewritten around them, with an `xcodebuild` build-gate sized for a busy CI box); custom agents *can* invoke custom agents (VS Code `agents:` is an allowlist); and **agent skills** (`.github/skills/`) — not prompt files — are the cross-surface equivalent of Claude Code skills. Every platform claim in v1.1.0 carries a verified-on date.
>
> **Purpose.** A reusable multi-agent orchestration setup for [GitHub Copilot](https://docs.github.com/en/copilot) tailored specifically for iOS engineering teams. Drops into any existing iOS codebase (UIKit / SwiftUI / Combine / async-await / Objective-C bridges) in 2-4 hours. Includes iOS-specialist agents, iOS-flavored instruction templates, an iOS pitfalls catalog, and a curated MCP integration list for the iOS toolchain.

This is **a sibling to**, not a replacement for, the stack-agnostic [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework). If your project is web, backend, or polyglot, use that one. If you ship to the App Store, this one is built for you.

---

## Why an iOS-only framework

The general Copilot framework is stack-agnostic by design — but its examples, specialist names, and bootstrap prompts lean web (Next.js, REST APIs, Stripe). An iOS adopter has to mentally translate `frontend-ui` → `ios-ui`, `npm run build` → `xcodebuild`, `next/image` rules → SwiftUI rendering rules, and so on. That works, but it leaks effort.

This framework eliminates that translation. Out of the box:

- **iOS specialist roster** — `ios-ui`, `ios-data`, `ios-network`, `ios-tests`, `ios-release`, `ios-privacy` (REVIEW-ONLY), `ios-perf`, `ios-bg`. Each one has a specific scope, tool allowlist, and Definition of Done.
- **iOS placeholders pre-filled** — `xcodebuild` build commands, Swift / Objective-C globs, `Info.plist` paths, `*.xcconfig` rules, code-signing scopes, App Store Connect references.
- **iOS pitfalls catalog** — 40 recurring gotchas (background `viewContext.save()`, `@MainActor` capture leaks, Info.plist usage descriptions, code signing rotation, the APNs entitlement, the one-shot notification prompt, the `NSURLSession` cookie jar, etc.) plus the framework lessons learned in production.
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
├── docs/                                  ← framework documentation (13 chapters)
│   ├── 00-QUICKSTART.md                   ← START HERE: step-by-step onboarding for any iOS project, incl. app + API repos
│   ├── 00-QUICKSTART.html                ← the same guide as one offline page with tabs for all three editions (open in a browser)
│   ├── 01-PRINCIPLES.md                   ← seven core principles
│   ├── 02-ARCHITECTURE.md                 ← .github/ (agents / instructions / skills / hooks) + iOS-flavored docs/ai-context/ layout
│   ├── 03-IOS-SPECIALISTS-GUIDE.md        ← the iOS specialist roster + scope tables; .agent.md + agents: allowlist
│   ├── 04-HANDOFF-SCHEMA.md               ← bidirectional schema with iOS examples (+ v1.1 cross-repo fields)
│   ├── 05-INSTRUCTIONS-AND-PROMPTS.md     ← path-globbed instructions, agent skills (cross-surface), prompt files (IDE-only)
│   ├── 06-INVOCATION-MODES.md             ← Chat / Code Review / Cloud Agent / headless `copilot -p` + the surface matrix
│   ├── 07-FOLDER-STRUCTURE.md             ← three-tier docs organization
│   ├── 08-IOS-COMMON-PITFALLS.md          ← 40 lessons: framework + iOS-specific (22 from v1.0, 18 new in v1.1)
│   ├── 09-IOS-RUNBOOK.md                  ← step-by-step iOS bootstrap (~2-4 hours)
│   ├── 10-MECHANICAL-ENFORCEMENT.md       ← (rewritten v1.1) Copilot hooks: the contract, five patterns, the xcodebuild build-gate, eleven design rules
│   ├── 11-IOS-MCP-CATALOG.md              ← curated iOS MCP servers + when to install each
│   ├── 12-PROJECT-TRUTH-AND-LEARNINGS.md  ← (v1.1) PROJECT.md / LEARNINGS.md / backlogs, the evidence ladder, the six-gate playbook
│   └── 13-MULTI-REPO-WORKSPACES.md        ← (v1.1) one iOS app + N API repos: layers, three delegation mechanisms, contracts, shared Swift packages
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
    ├── PROJECT.md.template · LEARNINGS.md.template · BACKLOG.md.template · GLOSSARY.md.template   ← (v1.1) the project-truth set
    ├── engineering-playbook-skill.md.template     ← (v1.1) six gates + evidence ladder, as .github/skills/<slug>-engineering/SKILL.md
    │
    ├── agents/                                    ← each becomes .github/agents/<name>.agent.md
    │   ├── orchestrator-agent.md.template         ← carries agents: [...] (VS Code subagent allowlist)
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
    ├── skills/                                    ← (v1.1) cross-surface Agent Skills — cloud agent, CLI and every IDE
    │   ├── commit-push-pr/SKILL.md.template       ← /commit-push-pr (xcodebuild gate, signing-material refusal)
    │   ├── verify-build/SKILL.md.template         ← /verify-build (xcodebuild; a killed build is inconclusive)
    │   └── correction-capture/SKILL.md.template   ← /correction-capture
    │
    ├── hooks/                                     ← (v1.1) hooks.json + hook-io / correction-detect / doc-freshness-track / lint-fix (swiftformat) / stop-gate (xcodebuild, 20-min cap)
    │
    ├── workspace/                                 ← (v1.1) the multi-repo layer: .code-workspace, manifest, router, orchestrator + 2 specialists, contract instructions, /delegate skill + scripts
    │
    └── prompts/                                   ← IDE-only prompt files (v1.0 forms, superseded by skills/)
        ├── prompt.md.template                     ← generic prompt-file shape
        ├── chatmode.md.template                   ← RETIRED — chat modes renamed to .agent.md; kept for migrations
        ├── correction-capture.prompt.md.template  ← /correction-capture (IDE-only form)
        ├── commit-push-pr.prompt.md.template      ← /commit-push-pr (IDE-only form)
        └── verify-build.prompt.md.template        ← /verify-build (IDE-only form)
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
ls .github/agents/                             # should list ios-ui.agent.md, ios-data.agent.md, etc.
ls .github/instructions/                       # should list domain-specific instruction files
ls .github/skills/                             # commit-push-pr, verify-build, correction-capture, <your-app>-engineering
ls docs/ai-context/                            # INDEX, PROJECT, LEARNINGS, GLOSSARY, HANDOFF_SCHEMA, …
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build  # should still succeed

# 8. Try a real task via the orchestrator
#    Type @<your-app>-orchestrator in Copilot Chat with a small bug or feature
#    Verify the handoff schema works end-to-end

# 9. Commit, PR, merge
git add .github/ docs/ai-context/ docs/_archive/ docs/*_BACKLOG.md
git commit -m "chore: bootstrap Copilot iOS orchestration framework"
git push -u origin setup/copilot-ios-framework
gh pr create --base develop --title "Bootstrap Copilot iOS orchestration"
```

Estimated time: **2-4 hours** (split across phases — see [`docs/09-IOS-RUNBOOK.md`](docs/09-IOS-RUNBOOK.md)).

### Scenario B — New iOS project (greenfield)

Skip INVENTORY-PROMPT (no existing config to scan). Paste BOOTSTRAP-PROMPT directly with your stack details. Estimated time: **45 min – 1 hour**.

### Scenario C — Just want to read the framework

Read in this order: README → [00-QUICKSTART](docs/00-QUICKSTART.md) → [01-PRINCIPLES](docs/01-PRINCIPLES.md) → [03-IOS-SPECIALISTS-GUIDE](docs/03-IOS-SPECIALISTS-GUIDE.md) → [09-IOS-RUNBOOK](docs/09-IOS-RUNBOOK.md). ~45 minutes.

---

## What this framework is opinionated about

- **iOS scope is the project's own definition of "iOS."** That includes universal apps (iOS + iPadOS), Mac Catalyst, watchOS / tvOS / visionOS companions if part of the same workspace, and iOS framework SDKs. Pure macOS / pure server-side Swift projects should use the [stack-agnostic framework](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) instead.
- **Universal evidence rule** — every claim Copilot makes (about a CoreData entity, a runtime version, an entitlement, an `Info.plist` key, an App Store guideline) must cite a file:line, schema, or rule. No vibes-based claims pass the orchestrator's validation.
- **REVIEW-ONLY for privacy + App Store compliance.** The `ios-privacy` agent has zero Write/Edit tools — it can read `Info.plist`, entitlements files, and privacy nutrition labels, but it cannot change them. That's a runtime lock, not a sticky note. (See [Principle 5](docs/01-PRINCIPLES.md#5-tools-are-runtime-enforced-everything-else-is-documentation-discipline).)
- **MCPs are recommendations, not requirements.** [`docs/11-IOS-MCP-CATALOG.md`](docs/11-IOS-MCP-CATALOG.md) lists what to consider; you install what you need. The framework works with zero MCPs installed (using xcodebuild directly through Bash) and gets sharper with each MCP added.

## What this framework does NOT include

- **iOS code samples or sample app.** Templates have placeholders. You fill in for your project.
- **Vendor lock-in.** No App Store accounts, no third-party SaaS dependencies. Only the documented Copilot customization surface (`.github/copilot-instructions.md`, `.github/instructions/`, `.github/agents/*.agent.md`, `.github/skills/`, `.github/hooks/`, and the IDE-only `.github/prompts/`; chat modes are retired — verified 2026-08-22) plus optional MCP integrations.
- **Hooks in the default install.** Copilot *does* have lifecycle hooks as of v1.1 (`.github/hooks/*.json`), and five iOS-pre-filled templates ship — but they are an explicit later-phase decision ([`docs/10-MECHANICAL-ENFORCEMENT.md`](docs/10-MECHANICAL-ENFORCEMENT.md)), not a setup-time default. Documentation discipline + `applyTo:` auto-loading stay the primary layer. (v1.0 said "Copilot doesn't have programmable lifecycle hooks" — retracted; Pitfall 19 in the [common pitfalls doc](docs/08-IOS-COMMON-PITFALLS.md).)
- **A proven iOS field trial.** Still none (see the changelog's known limitations) — the multi-repo layer in particular is designed from verified platform behaviour, not from an iOS adoption.

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
- **Per quarter:** run [`prompts/REFINEMENT-PROMPT.md`](prompts/REFINEMENT-PROMPT.md) to audit specialist scope drift, archive stale docs, decide if MCPs should be added/removed, re-check platform drift (Pitfall 23 — three v1.0 claims were false within three months), and re-stamp `PROJECT.md` §3
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

**Q: Prompt files or skills?**
Skills (`.github/skills/<name>/SKILL.md`). They work on the cloud agent, the CLI, code review and every IDE; prompt files are IDE-only and the VS Code docs say to convert a prompt to a skill for the Agent Host. The v1.0 prompt-file templates (`/commit-push-pr`, `/verify-build`, `/correction-capture`) are kept for IDE use; the v1.1 skills supersede them — same `xcodebuild` pre-fills.

**Q: We have one iOS app repo plus several API repos (and an Android repo). Agents at every level, or one top-level orchestrator?**
Both, in layers — and not a separate framework. Every repo keeps its own install (the iOS repo keeps this one); shared specialists move to the organisation's `.github-private/agents/`; a workspace repo with a `.code-workspace` file, a manifest and gitignored clones holds *only* the cross-repo orchestrator, the service map and the contract rules, and delegates writes to each child's own orchestrator (its own `copilot -p` session, or a VS Code subagent). The iOS app is a consumer of every contract; a shared Swift package is a contract too, pinned by tag. Design, verified platform behaviour, and a one-afternoon POC recipe: [`docs/13-MULTI-REPO-WORKSPACES.md`](docs/13-MULTI-REPO-WORKSPACES.md) + `templates/workspace/`.

**Q: Can I share with another iOS team?**
This repo is public, MIT licensed. Use freely.

## Companion frameworks

- [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) — the stack-agnostic parent. Use for non-iOS projects.
- [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) — the Claude Code equivalent. Use if your team uses Claude Code instead of (or alongside) Copilot.

All three editions released together on **2026-08-22**: Claude v1.2.0, Copilot v1.2.0, iOS v1.1.0 — the same production lessons (project truth, multi-repo workspaces, the new pitfalls) and the same platform re-verification, each in its own edition's vocabulary.

## Contributing

iOS adopters' real-world experience is the main feedback channel for v1.x. If you discover an iOS-specific pitfall, a missing specialist, a better instruction-file pattern, or an MCP worth adding — open an issue or PR. The framework improves as it sees more projects.

## License

MIT (see `LICENSE`). Use freely. Adapt freely. Attribution appreciated but not required.
