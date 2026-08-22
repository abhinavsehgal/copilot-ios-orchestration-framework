# 12 — Project Truth, Learnings, and the Evidence Ladder

> Added in v1.1.0, ported from the stack-agnostic Copilot edition's chapter 11. Distilled from three
> further months of running the framework on a production codebase with many parallel agent sessions.
> Everything here is domain agnostic — iOS-flavoured only where an example needed a concrete shape —
> and nothing in it depends on a Copilot surface: it applies to the cloud agent, IDE chat, and the
> CLI alike.

The v1.0 framework answered *"who does the work and how do they talk to each other?"* (agents,
handoff schema, instruction files). Three months of real use exposed the next failure class: **a
fresh agent with no transcript knows only what is in git — and git mostly recorded *code*, not
*truth*.**

Symptoms, all observed more than once:

- An agent built a feature that already existed, because nothing said "this exists, it is just
  hidden behind a flag".
- An agent treated a feature that had only reached an internal build as live on the App Store,
  because the orientation map described the feature but not *where it was deployed*.
- A "we'll do that later" spoken at the end of a session vanished, and was re-discovered as a bug
  two weeks on.
- An instruction file documented the wrong HTTP verb for an API route the app calls. Three agents
  trusted it. The fourth read the route.

The fix is three more documents, one playbook, and one enforced habit. This chapter specifies them.

---

## The three knowledge stores (and what goes where)

| Store | File | Answers | Written by | Loaded |
|---|---|---|---|---|
| **Project truth** | `docs/ai-context/PROJECT.md` | *What is true right now?* Environments, what is live where, sources of truth per domain, verified commands, definitions of done | Humans + agents, on every state change | Read at the start of every substantial task |
| **Learnings** | `docs/ai-context/LEARNINGS.md` | *What did we decide, what failed, what keeps going wrong, how has the owner corrected us?* | Agents, same turn a lesson emerges | Skim §D (corrections) before the first edit |
| **Backlogs** | `docs/<AREA>_BACKLOG.md` | *What did we defer, and when should we come back?* | Agents, before any turn that names deferred work ends | On "what's pending on X?" |

These sit alongside the stores v1.0 already had: `.github/copilot-instructions.md` (router —
or `AGENTS.md` if the project wants the router read by more than one AI tool),
`.github/instructions/*.instructions.md` (path-scoped hard rules, `applyTo:`),
`docs/ai-context/<area>.md` (orientation), `docs/<UPPERCASE>.md` (canonical reference), and any
IDE- or CLI-local memory the tool keeps (not visible to teammates).

**One home per fact.** Every other place links to it. When the same rule turns up in two files,
consolidate and leave a pointer. The single most corrosive doc smell is two copies that drift.

### Where a new piece of knowledge goes

| Kind of knowledge | Home |
|---|---|
| Universal behaviour + routing | `.github/copilot-instructions.md` (keep under ~200 lines) |
| Current state — what is live where, flags, applied migrations | `PROJECT.md` §3 + `docs/CHANGELOG.md` |
| A decision and its rationale | `LEARNINGS.md` §A |
| An approach that failed or proved unsafe | `LEARNINGS.md` §B |
| A recurring bug: symptom → first thing to check | `LEARNINGS.md` §C |
| A correction the owner had to give more than once | `LEARNINGS.md` §D **and** an instruction file if path-scoped |
| A hard rule scoped to file paths | `.github/instructions/<domain>.instructions.md` (`applyTo:`) |
| Per-area orientation (50–150 lines) | `docs/ai-context/<area>.md` |
| Deferred work | `docs/<AREA>_BACKLOG.md` |
| A multi-step repeatable procedure | `.github/skills/<name>/SKILL.md` |
| Machine-private session pointers | Any IDE- or CLI-local memory the tool keeps — **not** visible to teammates or other agents; anything durable must *also* land in one of the above |

**Don't persist:** session-specific context, one-off data values, anything derivable from code or
git in seconds, generic engineering advice.

---

## `PROJECT.md` — the freshness contract

Template: `templates/PROJECT.md.template`. The sections that earn their keep:

1. **Product + roles** — one paragraph and a table of roles with their permission boundaries.
2. **Environments, branches, deploy** — and, critically, **what the deploy pipeline does NOT do**.
   On iOS the usual surprises: provisioning-profile rotation, App Store Connect metadata and
   screenshots, and TestFlight build promotion to a wider group or to App Review are **not** done by
   CI unless a Fastlane lane or an Xcode Cloud workflow explicitly does them; a backend migration the app depends on does not
   apply itself either. "Migrations don't auto-apply" cost a real outage.
3. **Current state — what is live where.** A table: feature → App Store / TestFlight / internal
   build / backend environment / flag / date. The environments an iOS app has are the **App Store
   build**, the **TestFlight build(s)** (external and internal groups), **internal / enterprise /
   ad-hoc builds**, and — for every feature that needs a server — the backend environment each build
   talks to. This is the single biggest fresh-agent trap: most recent work is usually on a TestFlight
   or internal build, flag-gated, and the App Store build is a version behind.
4. **Repository map** — verified, date-stamped.
5. **Sources of truth by domain** — the canonical helper / table / module for each concept, so an
   agent reuses instead of re-implementing.
6. **Verified commands** — copied from the real build/test config, with the date.
7. **Definitions of done per change type.**
8. **Known sharp edges + unverified items.**

**The contract:** every fact is date-stamped, and the header names the commit it was verified
against. If today is much later than the stamp, trust `docs/CHANGELOG.md` over the state table,
*then fix the table*. A state claim without a date becomes a lie the day after it stops being true.

---

## `LEARNINGS.md` — decisions, failures, corrections

Template: `templates/LEARNINGS.md.template`. Six sections:

- **A. Decision records** — what was decided, by whom, why, dated.
- **B. Failed or unsafe approaches** — so nobody re-tries them. Each entry: what was tried, what
  broke, what replaced it.
- **C. Recurring bug patterns** — symptom → first thing to check. The cheapest section to write and
  the one that saves the most hours.
- **D. Agent behaviour corrections** — written as *"Instead of: … / Do: …"* pairs. These came from
  real corrections; treat them as standing instructions.
- **E. Working with the project owner** — the owner's preferred formats and approval gates.
  Examples that generalise: *multi-item status = a table, not prose*; *open decisions = one numbered
  list, resolved one at a time, never drop an item*; *approval is per-incident and current-turn,
  never carried forward from an earlier session*.
- **F. Documentation maintenance protocol** — the "where does knowledge go" table above, plus:
  date-stamp state claims; prune on contact (a verified-stale entry is fixed or deleted in the same
  turn, never left); unresolved conflicts get a ⚠ CONFLICT flag for the owner, never a silent pick.

Where a lesson is already a hard rule, `LEARNINGS.md` carries one line and a pointer — never a copy.

---

## Backlogs — deferred work must be written, not spoken

If an end-of-turn summary names *any* "we'll do this later", follow-up PR, known gap, or deferred
phase, the agent **must** append it to `docs/<AREA>_BACKLOG.md` before the turn ends.
Template: `templates/BACKLOG.md.template`. Each entry needs:

```markdown
### <Short title>
- **Why.** (one sentence on user value or technical reason)
- **Effort.** (days/weeks)
- **Where.** (file paths or component names if known)
- **Revisit when.** (the condition or signal to look for)
```

When done, prepend `✓ DONE — PR #N (date)` to the title and leave the body in place for context.

Two corollaries learned the hard way:

- **A backlog card is a claim, not evidence.** "Done" on a card proves nothing about production;
  verify state in `PROJECT.md` §3 or the deployed system.
- **Backlog ids collide across concurrent sessions.** Before taking the next `AREA-N` id (or the
  next migration number), check the remote *and* other open PRs. Two sessions took the same number
  on the same day, twice.

---

## Every production push freshens the docs — same turn, no exceptions

The moment a change lands on the production branch — or a build is submitted to App Review or
promoted to an external TestFlight group — a future agent sees only what is in git. So a production
push must be paired, **in the same turn**, with updates to the full doc set for what shipped:

1. `docs/CHANGELOG.md` — the mandatory anchor (a release entry).
2. The affected `docs/ai-context/<area>.md` orientation map(s).
3. Any affected `.github/instructions/*.instructions.md` hard rules.
4. `PROJECT.md` §3 if the production *state* changed (new feature in the App Store build, a flag
   flip, a TestFlight promotion, a backend migration applied).

Documentation discipline alone skipped this under deadline pressure roughly one push in five. It is
now an `agentStop` hook (Chapter 10, Pattern 5 — `docs/10-MECHANICAL-ENFORCEMENT.md`) which refuses
to end a turn in which a production push happened and `docs/CHANGELOG.md` was not touched
afterwards. Teams without hooks encode the same list in the orchestrator's Definition of Done and
in the `/commit-push-pr`-style prompt or skill.

---

## The engineering playbook — six gates and the evidence ladder

Template: `templates/engineering-playbook-skill.md.template`, installed as an Agent Skill at
`.github/skills/<project-slug>-engineering/SKILL.md`. A project-wide skill that every substantial
task runs through; the per-task-type skills (investigate-bug, build-feature) encode the checklists,
this one encodes the gates they share. **If a gate conflicts with a hard rule, the rule wins.**

| Gate | What it forces |
|---|---|
| 1 Scope | Restate the user-visible outcome; classify (bug / extension / pivot / data correction / migration / ops / docs); name affected roles and blast radius; say what is *out* of scope |
| 2 Evidence | Read the minimum routed context; **audit what exists before building** (already supported / supported-but-broken / partial / not supported / not needed); trace the real read/write path; tag every claim with a confidence class |
| 3 Design challenge | Smallest viable change + one rejected alternative; find the existing pattern to mirror; state the failure condition; run the standing impact list (permissions, sensitive data, every client, accessibility, rollout compatibility, flag gating, money paths) |
| 4 Implement | Narrow diffs; new user-visible behaviour lands flag-gated default-off; tests at the right level; code + migrations + scripts + docs in the same change |
| 5 Verify | The evidence ladder below — a green build is evidence, not proof |
| 6 Report | Files, instruction files read, checks run **with actual results**, risks and what was *not* verified (with its confidence class), docs updated, follow-ups written to a backlog |

### The evidence-confidence taxonomy

Mandatory in plans and reports. Nothing below `verified-*` may silently become a premise.

| Class | Meaning |
|---|---|
| `verified-code` / `verified-schema` / `verified-test` / `verified-git` | You looked at the thing itself |
| `documented-unverified` | A doc says so — including *this project's own instruction files* — and you didn't check |
| `historical` | True at a past date; may be stale |
| `unknown` | Needs confirmation |

### The proof ladder (in order of authority)

1. **The user-visible flow, exercised** — drive the real screen, capture the result. UI claims need
   UI proof, never code-reading. Hard-to-reach states are *created* (test accounts, seeds), not
   skipped.
2. **The data, landed** — query for the rows the change was supposed to produce. Proxy signals
   ("queued", "in scope", "deployed", "nothing to do") prove nothing.
3. **The third party, confirmed** — if an external service is involved, check its dashboard or API,
   not your code's return value.
4. **Regression edges** — adjacent tests, permission boundaries, empty/error/loading states, *both*
   sides of any two-sided flow.

Two rules that belong to the ladder:

- **Verify by reading the row, not by reasoning.** Two confident mechanisms were predicted for one
  write; both were wrong; a single query settled it in seconds.
- **Never report a negative result from a reader you have not verified can see the whole set.**
  Default pagination caps (ORMs, REST layers, search APIs) silently truncate. A "none found" from a
  truncating reader is not evidence — check the reader's ceiling first.

---

## Multi-client parity (when more than one client consumes one backend)

A second client is an unusually good audit of the first, and the source of a whole class of drift.
These rules are generic to any web + mobile, or any two consumers of one API, and they are what
makes Chapter 13's cross-repo contracts workable. **An iOS app is almost always a CONSUMER** — of
one or more API repos it does not own, often alongside a web client and an Android app consuming
the same endpoints — which is exactly why the multi-repo layer in Chapter 13 exists: the contract
the app pins (an OpenAPI or GraphQL spec, or a shared Swift package) lives somewhere else, and a
field renamed there fails silently here.

- **One client is the functional source of truth; the others may differ in presentation only.**
  What a person can *see and do* may not differ between clients. Layout, density, device affordances
  may. A deliberate divergence is written down with its reason.
- **Same heading ⇒ same endpoint.** Before building a panel that mirrors another client's panel,
  open that client's component and find what it actually *fetches*. Two cards with the same title
  and the same chart fed from two different endpoints showed one user two different answers to one
  question. Record the panel → source pairing in the orientation map.
- **One brain.** Business logic lives server-side; clients render. If a client needs derived data
  another client computes locally, extend the API rather than porting the computation.
- **Changing a contract? Grep every consumer in the same turn.** Clients pin response shapes; a
  renamed field fails silently at runtime (fetch succeeds, screen is empty). Keep a **mirror
  registry** — a table of code deliberately duplicated across clients (palette, label mappers,
  status derivations) — and change both sides in one PR.
- **Never ship a client that hard-depends on a brand-new server route.** Clients and servers deploy
  on different clocks, and an installed client can be older *or newer* than the server. Probe or
  degrade, and say so on screen when degraded.

---

## Smaller lessons that earned a line

- **Search before building.** Grep for the feature first. An existing implementation was rebuilt
  because nothing advertised it — and a "hide until ready" guard had silently removed it for a
  week. Guards that hide features need a visible "hidden because" signal somewhere.
- **Diagnostic scaffolding is removed in the same change as the fix.** An on-screen error line
  that cracked a bug becomes a permanent leak of raw technical text if left in.
- **Two words for one thing ships bugs.** Ambiguous vocabulary (the same word meaning two things in
  two subsystems) caused a silent filter miss in production. Keep a glossary in
  `docs/ai-context/GLOSSARY.md`; adding a fifth name for an existing concept is a defect, not style.
- **Corrections harden into instruction files, never into "I'll remember".** Any memory the IDE or
  CLI keeps is machine-local and invisible to every other agent and teammate.

---

## Cross-links

- `docs/10-MECHANICAL-ENFORCEMENT.md` — Pattern 5 (doc-freshness gate, `agentStop`) enforces the
  production-push rule.
- `docs/13-MULTI-REPO-WORKSPACES.md` — the parity rules above applied across repositories.
- `docs/08-IOS-COMMON-PITFALLS.md` — Pitfalls 23–31 are the incident reports behind this chapter.
- `templates/PROJECT.md.template`, `templates/LEARNINGS.md.template`,
  `templates/BACKLOG.md.template`, `templates/GLOSSARY.md.template`,
  `templates/engineering-playbook-skill.md.template`.
