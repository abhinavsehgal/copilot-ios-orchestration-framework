# 11 — iOS MCP Catalog

A curated list of MCP (Model Context Protocol) servers worth wiring up to Copilot for iOS work. Each entry has a "what it's for / when to install / minimum tier / verification" rubric so you can pick the smallest useful set.

> **Note on the MCP ecosystem.** The Copilot MCP marketplace and community-published MCPs evolve quickly. **Always check the marketplace + the MCP's own README before installing.** Names, packages, and APIs in the table below were accurate as of v1.0 release; some may have moved or been superseded by the time you read this. Where I'm not 100% sure of an MCP's current status, I'll mark it `[verify before install]`.
>
> Adopters who find newer / better MCPs: please open a PR to update this catalog.

## How MCPs interact with this framework

Each agent in `.github/agents/` declares an `mcp-servers:` allowlist:

```yaml
---
name: ios-release
description: Fastlane / TestFlight / App Store Connect specialist
tools: Read, Edit, MultiEdit, Bash, Grep, Glob, mcp__app-store-connect__*, mcp__github__*
mcp-servers: app-store-connect, github
target: vscode, github-copilot
---
```

The harness restricts the agent to MCPs in `mcp-servers:`. So `ios-release` can hit App Store Connect; `ios-ui` can't (because `ios-ui`'s allowlist doesn't include it). Defense in depth.

## Recommended minimum set (zero MCPs)

The framework works with **no MCPs at all**. All iOS toolchain access happens via `Bash` shell-out:

```bash
# Build
xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build

# Test
xcodebuild test -workspace MyApp.xcworkspace -scheme MyApp -destination 'platform=iOS Simulator,name=iPhone 15'

# Simulator control
xcrun simctl boot "iPhone 15"
xcrun simctl install booted /path/to/MyApp.app
xcrun simctl launch booted com.example.MyApp

# Logs
xcrun simctl spawn booted log stream --predicate 'subsystem == "com.example.MyApp"'

# Provisioning
security find-identity -p codesigning -v
```

This is the minimum viable setup. MCPs amplify productivity but aren't required.

## Catalog (recommended order of adoption)

### Tier 1 — install first, biggest ROI

#### **GitHub MCP**
Status: Official, GA. Works on every Copilot surface.

| | |
|---|---|
| What it's for | PR / issue / release / branch operations from inside Chat |
| When to install | Immediately, every iOS team |
| Used by agents | All — every specialist may want to query PR state, file an issue, list recent commits |
| Verification | After install, ask Copilot Chat *"list open PRs assigned to me"* — should return real data |
| Min Copilot tier | Pro / Business / Enterprise (custom agents requirement) |

#### **Xcode / xcodebuild MCP** `[verify before install]`
There are several community MCPs that wrap `xcodebuild` and `simctl`. As of v1.0 release, the most popular is one of:

- `xcodebuild-mcp` (community, bun/npm)
- An iOS-specific MCP shipped via the Copilot marketplace under the "iOS Toolchain" category

| | |
|---|---|
| What it's for | Run xcodebuild / list schemes / boot simulators / install app / capture logs without leaving Chat |
| When to install | Day 2 of bootstrap, after you've verified the framework basics |
| Used by agents | `ios-release`, `ios-tests`, `ios-perf` |
| Verification | Ask Copilot *"build the MyApp scheme for iPhone 15 simulator and report any errors"* — should run xcodebuild and surface the result |
| Min Copilot tier | Pro / Business / Enterprise |

If no Xcode MCP is available in your registry, fall back to Bash shell-out with `xcodebuild`. Same outcome, slightly more verbose.

#### **Mobile UI Automation MCP** (mobile-mcp / similar)
Status: `[verify before install]` — community, widely used.

Reference: [mobile-next/mobile-mcp](https://github.com/mobile-next/mobile-mcp) is one popular implementation covering iOS Simulator + Android.

| | |
|---|---|
| What it's for | Drive a simulator: tap, swipe, screenshot, accessibility tree dump. Useful for QA + reproducing UI bugs |
| When to install | When `ios-tests` is doing real UI test authoring or `ios-perf` is reproducing layout bugs |
| Used by agents | `ios-tests`, `ios-perf`, `ios-ui` (for accessibility audit) |
| Verification | Boot a simulator, install your app, then ask Copilot *"tap the 'Sign In' button and screenshot"* |
| Min Copilot tier | Pro / Business / Enterprise |

### Tier 2 — install for specific specialists

#### **App Store Connect API MCP** `[verify before install]`
Status: community implementations exist; the official ASC API key route is well-documented even without an MCP.

| | |
|---|---|
| What it's for | Query TestFlight builds, list app metadata, fetch crash reports, review status |
| When to install | Once `ios-release` is doing regular submission work and Fastlane already handles the heavy lift |
| Used by agents | `ios-release`, `ios-perf` (for crash report fetch) |
| Verification | Ask Copilot *"list the 5 most recent TestFlight builds"* — should return real ASC data |
| Min Copilot tier | Pro / Business / Enterprise; requires App Store Connect API key configured |

If no MCP exists, shell out to Fastlane:
```bash
fastlane run latest_testflight_build_number app_identifier:com.example.MyApp
```

#### **Sentry MCP**
Status: Official, GA.

| | |
|---|---|
| What it's for | Pull crash / error reports from Sentry directly into Chat for triage |
| When to install | When you have Sentry instrumented and `ios-perf` is doing crash triage |
| Used by agents | `ios-perf`, `ios-release` (post-release crash watch) |
| Verification | Ask Copilot *"show the top 5 unresolved iOS crashes from the last 7 days"* |
| Min Copilot tier | Pro / Business / Enterprise; requires Sentry API token |

#### **Firebase / Crashlytics MCP** `[verify before install]`
Status: community implementations exist; check current ecosystem.

| | |
|---|---|
| What it's for | Pull Crashlytics reports, Firebase Analytics events, Remote Config values |
| When to install | If your stack is Firebase-heavy and you don't already have Sentry |
| Used by agents | `ios-perf`, `ios-release` |
| Verification | Ask Copilot *"top 3 Crashlytics issues last release"* |
| Min Copilot tier | Pro / Business / Enterprise |

### Tier 3 — install for narrow workflows

#### **Slack MCP**
Status: Official, GA.

| | |
|---|---|
| What it's for | Search team Slack for discussions, post into release channel, get bug-thread context |
| When to install | When the team uses Slack for release announcements / bug triage |
| Used by agents | `ios-release` (release-channel posts), orchestrator (bug-thread context fetch) |
| Verification | Ask Copilot *"find recent #ios-bugs threads about login crashes"* |
| Min Copilot tier | Pro / Business / Enterprise |

#### **Notion / Linear / Jira MCP**
Status: Official MCPs exist for each.

| | |
|---|---|
| What it's for | Read / write tickets and project docs directly from Chat |
| When to install | If your team's product spec / bug tracker is one of these |
| Used by agents | Orchestrator (linking PRs to tickets), `ios-release` (release-notes generation from completed tickets) |
| Verification | Ask Copilot *"show all iOS-tagged tickets in this sprint"* |
| Min Copilot tier | Pro / Business / Enterprise |

#### **Charles / Proxyman MCP** `[verify before install]`
Status: Likely community-only; may not exist as a dedicated MCP.

| | |
|---|---|
| What it's for | Inspect HTTP traffic from a simulator / device for `ios-network` debugging |
| When to install | Almost never — this is usually a manual debug session, not an automated workflow |
| Alternative | Use the Charles / Proxyman desktop app directly, copy interesting traffic into Chat |

### Tier 4 — interesting but rare

These exist or might exist; install only if your project has a specific need:

- **fastlane MCP** — wraps fastlane lanes; alternative to Bash shell-out
- **TestFlight automation MCP** — schedule beta builds; usually overkill (Fastlane handles it)
- **HealthKit / iCloud / CloudKit MCPs** — for apps deeply tied to those frameworks
- **MapKit MCP** — for location-heavy apps
- **AppCenter MCP** — if you still use App Center for distribution

## How to discover what's actually available

1. **GitHub Copilot MCP marketplace** — open Copilot Chat, type `/mcp` (if available in your version) or check Copilot settings for MCP browser
2. **GitHub topics:** [github.com/topics/mcp-server](https://github.com/topics/mcp-server) — community-maintained MCP servers
3. **awesome-mcp lists** on GitHub — curated by the MCP community

## Per-agent MCP allowlist examples

Putting it together — what each iOS specialist's `mcp-servers:` field looks like in a project that has installed Tier 1 + a couple of Tier 2 MCPs:

```yaml
# .github/agents/<your-app>-orchestrator.md
mcp-servers: github

# .github/agents/ios-ui.md
mcp-servers: mobile-ui-automation

# .github/agents/ios-data.md
# (none — pure file-system work; let it use Bash for sqlite if needed)

# .github/agents/ios-network.md
mcp-servers: github

# .github/agents/ios-tests.md
mcp-servers: xcodebuild, mobile-ui-automation

# .github/agents/ios-release.md
mcp-servers: xcodebuild, app-store-connect, github

# .github/agents/ios-privacy.md
# REVIEW-ONLY — no MCPs needed; pure read access

# .github/agents/ios-perf.md
mcp-servers: xcodebuild, sentry, app-store-connect

# .github/agents/ios-bg.md
mcp-servers: xcodebuild
```

## Adoption guidance

- **Don't install all Tier 1 MCPs at bootstrap time.** Install GitHub first; verify it works; install Xcode/simctl MCP next; verify; install mobile-mcp last. Three separate validation passes catches problems early.
- **Pin MCP versions** in your team's MCP config if your registry supports it. Auto-updates have surprised teams when a new MCP version changed the tool names and broke `mcp-servers:` allowlists.
- **Rotate API tokens** for App Store Connect / Sentry / etc. on a 90-day cadence. The MCP's auth lives in your Copilot config; rotating means updating both ASC and the MCP config.
- **Audit MCP scope quarterly** as part of REFINEMENT-PROMPT. If `ios-perf` is allowed `app-store-connect` but never uses it, drop it from the allowlist.

## Cross-links

- [`02-ARCHITECTURE.md`](02-ARCHITECTURE.md) — the `.github/agents/<NAME>.md` shape including `mcp-servers:`
- [`03-IOS-SPECIALISTS-GUIDE.md`](03-IOS-SPECIALISTS-GUIDE.md) — what each specialist needs from MCPs
- [`prompts/REFINEMENT-PROMPT.md`](../prompts/REFINEMENT-PROMPT.md) — the quarterly audit that re-evaluates MCP allowlist drift
