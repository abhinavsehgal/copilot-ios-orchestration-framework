# INVENTORY-PROMPT (iOS-anchored)

> **Paste this entire file into Copilot Chat. Replace `<framework path>` with the absolute local path to this repo on your machine. The prompt runs read-only — it scans your iOS project and proposes a framework setup. It does NOT write any files.**

You are running an inventory pass for the [Copilot iOS Orchestration Framework](https://github.com/abhinavsehgal/copilot-ios-orchestration-framework). Your job is to **read-only scan this iOS project** and produce a framework-setup proposal. You will NOT generate any files in this pass — only propose what should be generated.

## Anchor — this is an iOS project

This is an **iOS application or iOS framework SDK** (UIKit / SwiftUI / Combine / async-await / Objective-C bridges as relevant). Do not propose web-flavored specialists like `frontend-ui` or `backend-api-db` — propose iOS-shaped specialists from the framework's default roster (`ios-ui`, `ios-data`, `ios-network`, `ios-tests`, `ios-release`, `ios-privacy`, `ios-perf`, `ios-bg`).

If you find evidence the project is NOT primarily iOS (e.g. it's a backend, web app, or pure cross-platform server) — STOP and tell the user, recommend the [stack-agnostic framework](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) instead.

## Step 1 — scan the project

Look for:

**iOS workspace / project signals:**
- `*.xcworkspace` directory
- `*.xcodeproj` directory
- `Podfile` and/or `Podfile.lock`
- `Package.swift` and/or `Package.resolved`
- `Cartfile` (rare in 2026)
- `Tuist/` directory (project generation)
- `fastlane/` directory
- `.swiftlint.yml` / `.swiftformat`

**Source layout:**
- Where Swift files live (`Sources/`, `App/`, top-level by group, etc.)
- Whether Objective-C exists (`*.h`, `*.m`, `*.mm`)
- Whether Interface Builder is used (`*.xib`, `*.storyboard`)
- Whether SwiftUI / UIKit / hybrid (heuristic: count `View {` declarations vs `UIViewController` subclasses)
- Whether multiple platforms in one workspace (watchOS / tvOS / visionOS targets)

**Persistence + framework signals:**
- `*.xcdatamodeld` (CoreData)
- Imports of `SwiftData`, `RealmSwift`, `GRDB`, `SQLite.swift`
- Custom file-storage code (Documents/Caches access patterns)

**Networking / API signals:**
- `URLSession` usage
- Imports of `Alamofire`, `Moya`, `Apollo`, `swift-openapi`
- Custom auth refresh / Keychain code

**Existing Copilot config:**
- `.github/copilot-instructions.md` (size + last modified)
- `.github/instructions/*.instructions.md`
- `.github/skills/*/SKILL.md`
- `.github/prompts/*.prompt.md`
- `.github/agents/*.agent.md` (and bare `*.md`)
- `.github/hooks/*.json`
- `.github/chatmodes/*.chatmode.md` (retired — flag for rename to `.agent.md`)
- `docs/ai-context/PROJECT.md`, `LEARNINGS.md`, `GLOSSARY.md`, `docs/*_BACKLOG.md` (does a project-truth set already exist?)

**Documentation tier signals:**
- `docs/` contents — count files, list them
- `docs/ai-context/` (does this already exist?)
- `docs/_archive/` (does this already exist?)

**Branching + release signals:**
- Default branch (`develop` / `main` / something else)
- Release branches (`release/*`?)
- `fastlane/Fastfile` lanes
- CI config (`.github/workflows/*.yml`, `.bitrise.yml`, `ci_scripts/` for Xcode Cloud)

## Step 2 — produce the proposal

Output the following in markdown, in this exact structure:

### Proposed framework setup

**Project identity:**
- Name (from Xcode scheme or project name)
- Slug (kebab-case version, used as orchestrator prefix)
- iOS deployment target (from project settings)
- Stack summary (1 sentence)

**Recommended specialist roster** (from the default 8, plus custom additions if signals warrant):

| Specialist | Include? | Reason |
|---|---|---|
| `<your-app>-orchestrator` | Yes | Required |
| `ios-ui` | Yes/No | (justify) |
| `ios-data` | Yes/No | (justify based on CoreData/SwiftData/Realm signals) |
| `ios-network` | Yes/No | (justify) |
| `ios-tests` | Yes/No | (justify based on test target presence) |
| `ios-release` | Yes/No | (justify based on Fastlane / CI presence) |
| `ios-privacy` | Yes/No | (justify based on Info.plist size / entitlements / IDFA / privacy manifest signals) |
| `ios-perf` | Yes/No | (justify based on app maturity) |
| `ios-bg` | Yes/No | (justify based on background mode entitlements / push setup / BGTaskScheduler usage) |

Plus any custom additions you spotted (e.g. `ios-watch` if there's a watchOS target, `ios-iap` if StoreKit-heavy, etc.).

**Proposed `applyTo:` globs** for each instruction file:

| Instruction file | applyTo: glob | Real files matched (sample) |
|---|---|---|
| `swiftui.instructions.md` | (your proposed glob) | (3 example real paths) |
| `concurrency.instructions.md` | (glob) | (3 examples) |
| `coredata.instructions.md` | (glob) | (3 examples or "no matches — drop this file") |
| `networking.instructions.md` | (glob) | (3 examples) |
| `info-plist.instructions.md` | (glob) | (path to Info.plist) |
| `code-signing.instructions.md` | (glob) | (paths to entitlements / xcconfig) |

**Existing Copilot config** (if any):
- `.github/copilot-instructions.md`: (exists / not exists; if exists, summarize content + flag drift)
- `.github/agents/`: (list existing agent names; flag any that conflict with proposed iOS specialists)
- `.github/instructions/`: (list existing instruction files; flag overlapping `applyTo:` globs)

If existing config is found: **flag clearly** that BOOTSTRAP-PROMPT will need pre-flight safety checks. Suggest the user back up `.github/` before running bootstrap.

**Tier-2 docs to author** (high-value `docs/<UPPERCASE>.md` files for this project):

Based on signals you found, recommend 3-5 tier-2 canonical docs to write (these go in `docs/`, not `docs/ai-context/`):
- `ARCHITECTURE.md` (always)
- `BUILD_SYSTEM.md` (if Tuist/Bazel/non-trivial Xcode setup)
- `PRIVACY_AND_DATA.md` (if privacy-sensitive data flows)
- `DEPENDENCIES.md` (if many Pods/SPM deps)
- `RELEASE_PROCESS.md` (if Fastlane lanes / multi-stage release)
- `INCIDENT_RESPONSE.md` (if mature production app)

### Open questions you must answer before BOOTSTRAP

Each question with a default if you have evidence to suggest one:

1. **Project name** — confirm or correct (default: `<your-detected-scheme>`)
2. **Default base branch** — `develop` / `main` / something else? (default: detected from git)
3. **iOS deployment target** — confirm (default: from Xcode project)
4. **Build command** — `xcodebuild` directly (`-workspace` or `-project`), via a Fastlane lane, or via an Xcode Cloud / CI script? (default: detected)
5. **Test command** — `xcodebuild test` or `fastlane test` or `xcodebuild test -only-testing:...`? (default: detected)
6. **Signing strategy** — Match / Xcode Cloud managed signing / manual / Xcode automatic? (default: detected from Fastlane or CI config if present)
7. **MCPs to install at bootstrap** — None (zero MCP setup), Tier 1 only (GitHub + xcodebuild), Tier 1+2 (add Sentry / ASC), or full Tier 1-3? (default: Tier 1 only)
8. **Specialists you want that aren't in my proposal** — anything missing?
9. **Specialists in my proposal you want to drop** — anything you don't need?

Wait for the user to answer all questions before they paste BOOTSTRAP-PROMPT.

### Sanity checks before BOOTSTRAP

After producing the proposal, also flag:

- ✓ / ✗ Test build passes (run `xcodebuild -workspace ... -scheme ... build` once and report)
- ✓ / ✗ git status clean (no uncommitted changes — bootstrap creates new files; you want a clean baseline)
- ✓ / ✗ Currently on a fresh setup branch (suggest `setup/copilot-ios-framework` if the user is still on `main`/`develop`)

### What this prompt is NOT going to do

- ❌ Not creating any files — output is proposal only
- ❌ Not installing any MCPs — flagging which to install is the user's decision
- ❌ Not modifying `.github/` — output is read-only

Output the entire proposal now. Wait for user feedback before BOOTSTRAP.
