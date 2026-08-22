# 07 — Folder Structure (three-tier docs)

How project documentation should organize itself for an iOS codebase using this framework.

## The three tiers

| Tier | Location | Purpose | Update frequency | Read by |
|---|---|---|---|---|
| **Orientation maps** | `docs/ai-context/<topic>.md` | 50-150 lines per area; gotchas; cross-links | Weekly to monthly (lots of churn) | Copilot agents (per task routing) |
| **Canonical references** | `docs/<UPPERCASE>.md` | Architecture, build system, privacy, dependencies — full detail | Quarterly to per-major-release | Humans + agents (when they need depth) |
| **Frozen archive** | `docs/_archive/<YYYY-MM>/<doc>.md` | Sprint reports, post-mortems, App Store rejection histories, snapshots | Append-only | Audits / retrospectives only — never linked from active docs |

## Why three tiers and not one

Without separation, `docs/` accumulates:
- Sprint reports from 2 years ago next to current architecture docs
- Outdated `UIWebView` migration notes treated as authoritative
- The post-mortem of a App Store reviewer rejection that was resolved a year ago
- HTML / PNG artifacts from a one-time presentation

After a year, a new iOS hire opens `docs/` and can't tell which docs are truth. Worse, Copilot reads them all and treats them all as authoritative — old docs cause drift.

The three-tier system makes it impossible to confuse active truth with frozen history. Active docs live in tiers 1 + 2. Tier 3 is one-way: things move in, never out.

## Tier 1 — orientation maps (`docs/ai-context/`)

These are short, opinionated, and Copilot-shaped. Each one is 50-150 lines and **scoped to one area** of your iOS app.

iOS examples:

| File | Covers |
|---|---|
| `INDEX.md` | The router — every task type → which orientation maps to read + which specialist to invoke |
| `PROJECT.md` | Current truth (v1.1): what is live on the App Store / TestFlight / backend, what CI does NOT do, sources of truth, verified commands — date-stamped ([`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md)) |
| `LEARNINGS.md` | Decisions, failed approaches, recurring bug patterns, agent corrections (v1.1) |
| `GLOSSARY.md` | One name per concept — a rule file, not a nicety (v1.1) |
| `swiftui-patterns.md` | Your project's SwiftUI conventions: state ownership, view composition, modifier order |
| `coredata-schema.md` | Schema versions, migration policy, where contexts live, naming conventions |
| `networking-architecture.md` | Endpoint definitions, auth refresh, retry/backoff, cert pinning policy |
| `release-process.md` | Branch → TestFlight → App Store flow, version-bumping policy, who approves what |
| `accessibility-baseline.md` | What "accessible enough to ship" means for this app (Dynamic Type XL, VoiceOver baseline) |
| `info-plist-policy.md` | Required usage descriptions, ATT prompt timing, third-party SDK manifests |
| `ORCHESTRATION_SPOONFEEDER.md` | Human-readable how-to-use for the team (copy-paste examples for skills, agent invocations) |
| `HANDOFF_SCHEMA.md` | The bidirectional contract (drop-in from `templates/HANDOFF_SCHEMA.md.template`) |

Each orientation map should:
- Open with a 3-sentence summary of the area
- List 3-7 hard gotchas specific to your project
- Cross-link to relevant `.github/instructions/*.instructions.md`
- Cross-link to the canonical doc (tier 2) for deeper detail

**They never duplicate canonical content.** They link.

## Tier 2 — canonical references (`docs/<UPPERCASE>.md`)

The deep detail. Long. Authoritative. Updated when architecture changes.

iOS examples:

| File | Purpose |
|---|---|
| `ARCHITECTURE.md` | Full architectural diagram + module dependency graph + history of major decisions |
| `BUILD_SYSTEM.md` | Xcode workspace layout, schemes, configurations, xcconfig overlays, signing strategy |
| `PRIVACY_AND_DATA.md` | What data is collected, where it's stored, how long, who has access (the App Privacy nutrition label backing document) |
| `DEPENDENCIES.md` | Every Pod / SPM dependency with rationale, last-audit date, who owns it |
| `CHANGELOG.md` | Per-release human-readable changes (separate from your release notes — internal-team detail) |
| `INCIDENT_RESPONSE.md` | What to do when production crashes spike or App Store reviewer rejects |
| `ON_CALL.md` | Rotation, escalation, who pages who |

Tier 2 docs are written for humans first, agents second. They're allowed to be long.

## Tier 3 — frozen archive (`docs/_archive/<YYYY-MM>/`)

Append-only. Per-month directories.

iOS examples:

```
docs/_archive/
├── 2025-08/
│   ├── ios-15-deprecation-plan.md         (no longer needed; iOS 16 is now min)
│   ├── post-mortem-app-store-rejection-1.0.4.md
│   └── sprint-12-summary.md
├── 2025-12/
│   ├── coredata-migration-v3-to-v4.md     (migration completed; superseded by current schema)
│   └── facebook-sdk-removal-runbook.md    (we removed Facebook SDK; runbook is historical)
└── 2026-04/
    └── visionos-companion-deprecation.md  (we shipped without it)
```

Rules for the archive:
- Never deleted (you might need it for an audit)
- Never linked from active docs (would cause drift)
- Never read by agents (would inject stale guidance)
- The archive's `README.md` (drop in from `templates/archive-README.md.template`) explicitly tells anyone who wanders in: *"This is frozen history. Do not treat as authoritative. If something here matters, promote it to a tier-1 or tier-2 doc."*

## Drift management

Every quarter, run `prompts/REFINEMENT-PROMPT.md`. It includes a tier-audit pass that:
1. Lists every file in `docs/ai-context/` and asks: is this still authoritative?
2. Lists every file in `docs/<UPPERCASE>.md` and asks: any drift since last update?
3. Proposes archive moves for things that are no longer current.
4. Surfaces broken cross-links.

A `context-librarian` specialist (one of the optional iOS specialists you can add) makes this their owned domain.

## Cross-links

- [`02-ARCHITECTURE.md`](02-ARCHITECTURE.md) — the `.github/` side of the same picture
- [`08-IOS-COMMON-PITFALLS.md`](08-IOS-COMMON-PITFALLS.md) § Pitfall 6 (*"Letting `docs/` rot without a librarian"*), § Pitfall 26 (a production push that does not freshen the docs)
- [`12-PROJECT-TRUTH-AND-LEARNINGS.md`](12-PROJECT-TRUTH-AND-LEARNINGS.md) — `PROJECT.md`, `LEARNINGS.md`, `GLOSSARY.md` and the per-area backlogs (`docs/<AREA>_BACKLOG.md`, tier 2)
- The `templates/INDEX.md.template`, `templates/SPOONFEEDER.md.template`, `templates/HANDOFF_SCHEMA.md.template`, `templates/archive-README.md.template`, and (v1.1) `templates/PROJECT.md.template`, `LEARNINGS.md.template`, `GLOSSARY.md.template`, `BACKLOG.md.template`
