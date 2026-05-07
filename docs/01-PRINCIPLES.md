# 01 — Principles

The seven principles this framework rests on. Every doc, prompt, and template implements these. The principles themselves are stack-agnostic; the **examples** here are iOS-flavored so you can see how they translate.

## 1. Orchestrator-worker pattern, documented not coded

One coordinator agent (`<your-app>-orchestrator`) reads tasks, classifies them, picks the right specialists (`ios-ui`, `ios-data`, `ios-network`, etc.), and aggregates findings. Specialists do focused work in their own context.

The orchestration is **prescriptive documentation**, not runtime code. The orchestrator is a markdown file at `.github/agents/<your-app>-orchestrator.md` with instructions; Copilot's harness provides the agent dropdown, tool restrictions, and target scoping. There is no orchestration service, no message queue, no central process.

This matters because:
- It works in every Copilot surface (VS Code, Xcode, JetBrains, Visual Studio, GitHub.com Chat, Cloud Agent)
- It's fully transparent — you can read every rule
- It's reversible — delete `.github/` and you're back to default Copilot
- Hard runtime enforcement (where it exists in Copilot — `tools:` allowlists, `target:` scopes) layers on top

## 2. Each specialist runs with focused tool scope

Every specialist agent declares a `tools:` allowlist in its frontmatter — and Copilot's harness enforces it. A `legal-compliance` or `ios-privacy` specialist that lists only `Read, Grep, Glob` **physically cannot** edit files. The Edit tool isn't in its allowlist, so the harness blocks the call.

This is the **single most important runtime defense** the framework provides. A REVIEW-ONLY specialist cannot hallucinate code into your repo because the harness blocks the tool call.

> **Note on context isolation:** Custom agents in Copilot Chat run within the active Chat session's context (unlike Claude Code subagents which run in fully isolated context windows). The Cloud Agent runs in true isolation. See [`06-INVOCATION-MODES.md`](06-INVOCATION-MODES.md) for the practical implications.

## 3. Universal evidence rule — "no evidence, no claim"

If a claim cannot be tied to a file:line, table:column, rule, doc anchor, test, or command output, it must be marked as an assumption (`confidence: low` outbound, or moved to `unverified_claims` inbound) and cannot be passed downstream as fact.

For iOS, "evidence" looks like:
- `Sources/Profile/AvatarView.swift:42` (file + line)
- `Info.plist:NSCameraUsageDescription` (Info.plist key)
- `Profile.xcdatamodeld/Profile.xcdatamodel/contents:User.streak` (CoreData entity:attribute)
- `Apple Human Interface Guidelines § Privacy` (HIG reference)
- `xcodebuild test -only-testing:ProfileTests/AvatarTests` output
- `App Store Review Guidelines § 5.1.1` (compliance citation)
- `git log --oneline -- Sources/Profile/` output

This rule appears verbatim in:
- The handoff schema (both directions)
- The orchestrator's "outbound discipline"
- Every specialist's "incoming handoff validation"
- Every path-globbed instruction file's preamble (the first thing Copilot reads when auto-loading the file)

It is the **core defense against cascading hallucinations** across hops. An orchestrator that hallucinates *"the bug is in `AvatarView.swift:42`"* must cite that path; the specialist re-verifies before editing; if wrong, the specialist returns `status: claim_rejected` with the evidence of what it actually found.

## 4. Failure condition — articulate what would prove you wrong

Every outbound handoff includes a `failure_condition` field: one sentence stating what observable evidence would prove the orchestrator's hypothesis or delegation premise wrong. If the specialist observes that condition, it stops, returns `claim_rejected`, and includes the triggering evidence.

iOS examples:

- `failure_condition: if the AvatarView.swift file has no @MainActor annotation, the orchestrator's hypothesis about a main-thread crash is wrong — stop and return claim_rejected`
- `failure_condition: if Info.plist already has NSPhotoLibraryUsageDescription, the orchestrator's "missing usage string" hypothesis is wrong — stop`
- `failure_condition: if the failing test is on iOS 16 only and the changeset doesn't touch any iOS 16-only API, the orchestrator's "regression in this PR" hypothesis is wrong — stop`

This is the **inverse of `verify_before_acting`**. Where `verify_before_acting` says "check these before acting," `failure_condition` says "if you observe this, STOP." Together they bracket the work in falsifiable claims.

## 5. Tools are runtime-enforced; everything else is documentation discipline

Copilot's harness enforces:
- `tools:` allowlists (specialist physically cannot use tools not listed in the agent file)
- `disable-model-invocation:` (prevents auto-routing to this agent)
- `user-invocable:` (controls manual selection)
- `target:` (scopes agent to specific environments — `vscode` or `github-copilot`)
- `mcp-servers:` per-agent MCP server scoping (so e.g. only the `ios-release` agent can call App Store Connect MCP)
- The `applyTo:` glob in `.github/instructions/<NAME>.instructions.md` (auto-loaded only when matching files are touched)
- Body length cap (~30,000 chars per file)

Everything else is **documentation discipline** — the agent follows its instructions because they're in the system prompt:
- Handoff schema fields being present
- Refusal of vague delegations
- Evidence rule being followed
- `failure_condition` observation rule
- Hop limits ("3rd same-specialist delegation → escalate to user")
- "Read the right instruction file before editing"
- Definition of Done completeness

Don't conflate the two layers. When you say *"the orchestrator can only delegate to project specialists,"* that's documentation in Copilot (no runtime allowlist for cross-agent invocation in current versions). When you say *"the `ios-privacy` agent cannot edit files,"* that IS runtime-enforced via the `tools:` field.

## 6. Three-tier documentation

iOS project documentation should split into three clearly-labeled tiers:

| Tier | Location | Purpose | Read by |
|---|---|---|---|
| Orientation maps | `docs/ai-context/<topic>.md` | 50-150 lines per area; iOS gotchas; cross-links | Copilot agents (per task routing) |
| Canonical references | `docs/<UPPERCASE>.md` | Architecture, ApprovedFlows, build-system, full detail | Humans + agents (when deeper) |
| Frozen archive | `docs/_archive/<date>/` | Sprint reports, post-mortems, App Store rejection logs, snapshots | Audits only — never linked from active docs |

This separation makes drift impossible. Active docs live in the first two tiers. The archive is append-only and never referenced by agents/instructions/active docs. When something is no longer authoritative (e.g. a deprecated `UIWebView`-based auth flow), it moves to archive.

## 7. Default Copilot stays default for inline completions

**Do not put the orchestrator's persona in `.github/copilot-instructions.md`.** That file is auto-loaded into EVERY Copilot interaction — including inline completions while you're typing Swift code. Loading the orchestrator's full persona there would:

- Slow down inline completions inside Xcode
- Pollute code-review prompts with delegation instructions
- Force every interaction through the orchestrator's "delegate everything" body

`.github/copilot-instructions.md` should be a **thin router** — golden rules + workflow + cross-links. The orchestrator agent is invoked explicitly when the user picks it from the agent dropdown or types `@<your-app>-orchestrator` in chat.

Three invocation modes coexist:
- **Default** — inline completions while writing Swift / typing in Xcode (uses repository-wide instructions only)
- **Specialist mode** — Chat with a specific specialist agent selected (e.g. `@ios-data` for a CoreData refactor)
- **Orchestrator mode** — Chat with `@<your-app>-orchestrator` for cross-domain work

See [`06-INVOCATION-MODES.md`](06-INVOCATION-MODES.md).

---

## What these principles get you

A team can work across Xcode, VS Code (with Swift Language Server), JetBrains AppCode (legacy), and on github.com (cloud agent, code review) with consistent behavior. Specialists never inherit baggage from unrelated work. Orchestrator decisions are auditable (every claim cites evidence — file:line, Info.plist key, App Store guideline). Hallucinations don't compound across hops. New iOS engineers see a clean `.github/` layout that tells them where to look — `.github/agents/ios-ui.md` for SwiftUI questions, `.github/instructions/coredata.instructions.md` auto-loads when editing CoreData files, etc.

The cost is upfront discipline: you spend a day writing the agent contracts, instruction files, and orientation maps. The payback is permanent — every future task benefits.
