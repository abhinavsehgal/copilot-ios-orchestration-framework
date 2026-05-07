# Changelog

All notable changes to the Copilot iOS Orchestration Framework. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] — 2026-05-07

### Initial public release

A standalone, iOS-focused fork of the architecture established in [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) v1.1.2.

### Added

- **Eleven-chapter doc set** covering the seven principles, architecture, iOS specialist roster, handoff schema with iOS examples, path-globbed instructions for iOS file layouts, invocation modes across IDEs + Cloud Agent, three-tier folder structure, twenty-plus iOS-specific common pitfalls, the iOS bootstrap runbook, mechanical enforcement (and why hooks don't apply on Copilot), and an iOS MCP integrations catalog.
- **Eleven iOS specialist agent templates** — orchestrator + `ios-ui` (SwiftUI / UIKit / accessibility) + `ios-data` (CoreData / SwiftData / Realm) + `ios-network` (URLSession / Combine / cert pinning) + `ios-tests` (XCTest / XCUITest / snapshots) + `ios-release` (Fastlane / TestFlight / ASC) + `ios-privacy` (REVIEW-ONLY: Info.plist / ATT / nutrition labels) + `ios-perf` (Instruments / hangs / launch time) + `ios-bg` (BGTaskScheduler / silent push) + generic review-only and specialist shapes for custom additions.
- **Six iOS instruction file templates** — SwiftUI lifecycle invariants, Swift concurrency rules, CoreData multi-context patterns, networking + cert pinning gotchas, Info.plist usage descriptions, code signing + provisioning rotation rules.
- **Three iOS-anchored bootstrap prompts** — INVENTORY (read-only iOS project scan), BOOTSTRAP (file generation with iOS pre-flight safety checks), REFINEMENT (post-bootstrap iOS audit).
- **Three iOS-pre-filled slash prompt templates** — `/correction-capture` (iOS-flavored), `/commit-push-pr` (xcodebuild + iOS git workflow pre-filled), `/verify-build` (xcodebuild + scheme detection pre-filled).
- **Curated iOS MCP integration catalog** at [`docs/11-IOS-MCP-CATALOG.md`](docs/11-IOS-MCP-CATALOG.md) — Xcode + simulator MCPs, mobile UI automation MCPs, App Store Connect API MCPs, Sentry / Firebase / Crashlytics MCPs, GitHub MCP, Slack MCP. Each entry has a "what it's for / when to install / minimum tier" rubric.
- **Pre-flight safety pass** for brownfield bootstrap — snapshot existing `.github/copilot-instructions.md`, naming-collision detection, `applyTo:` glob conflict detection, drift detection, decision gate before any file write.

### Provenance

The principles, handoff schema, three-tier docs structure, and bootstrap workflow originated in the parent stack-agnostic framework. This iOS framework specializes them: every example, every specialist scope, every path glob, every pitfall, every MCP recommendation is iOS-flavored from the start. No mental translation required.

### Why a separate framework instead of an iOS section in the parent

The parent framework's strength is being stack-agnostic. Loading it with iOS-specific specialist names, build commands, and pitfalls would make it noisier for web / backend adopters. iOS engineers benefit from a focused framework that "just knows" the toolchain — `xcodebuild` instead of `npm run build`, `Info.plist` instead of `next.config.ts`, `App Store Connect` instead of `Stripe webhooks`.

### Known limitations

- **No proven adoption yet on a real iOS codebase.** This is v1.0; iOS adopters' first-week feedback drives v1.0.1 / v1.1.0.
- **MCP catalog is a snapshot.** The MCP ecosystem moves fast. Catalog will need refreshes; adopters should always check their MCP registry before installing.
- **No watchOS / tvOS / visionOS-specific specialist templates yet.** If your app has heavy companion-platform code, you'll add `ios-watch` / `ios-tv` / `ios-vision` specialists in your project — the templates support that pattern, but no pre-built templates ship in v1.0.

---

## Companion releases

- [`github-copilot-orchestration-framework`](https://github.com/abhinavsehgal/github-copilot-orchestration-framework) — the stack-agnostic parent.
- [`claude-orchestration-framework`](https://github.com/abhinavsehgal/claude-orchestration-framework) — the Claude Code equivalent.

This framework is intentionally **at v1.0.0**, not v1.1.x — it's a separate maturation path. Future iOS-specific releases will track the iOS toolchain (new Xcode versions, new App Store policies, new SwiftUI APIs), independent of the parent framework's release cadence.
