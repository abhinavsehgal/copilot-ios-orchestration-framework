# 13 — Multi-Repo Workspaces (one iOS app + N backend repos)

> Added in v1.1.0, ported from the stack-agnostic Copilot edition's chapter 12. Platform facts
> verified against the official GitHub Copilot docs and the VS Code Copilot docs on 2026-08-22
> (custom agents, subagents, agent skills, hooks, Copilot CLI, coding agent, agent-customization
> overview). Re-verify before relying on a behaviour.

## The question this chapter answers

> *"We have the iOS app in one repo, the backend split across several API repos (plus an API
> gateway), a web repo, and an Android repo. Does orchestration need agents at every level, or can
> a top-level orchestrator understand all the services? Should we create a separate workspace repo
> with a workspace file and the agent files, gitignore the sub-projects, and skip CI at that level?"*

The canonical iOS shape is **one iOS app repo + N backend/API repos (+ maybe an Android repo and a
web repo)**. The iOS app is a **consumer**: it owns no contract, it pins the shape of every response
it reads, and it ships on a different clock from every producer (App Review adds days; an installed
build can be older *or newer* than the server). The contract between them is usually an OpenAPI /
GraphQL spec the API repo owns — or a shared Swift package (below).

Short answer: **three layers, each optional, each building on the one below.** The per-repo
framework is layer 1 and stays mandatory. Shared specialists move to the organisation's
`.github`/`.github-private` repo (layer 2) the moment two repos would otherwise carry copies. A
workspace repo (layer 3) — a `.code-workspace` file plus the cross-repo orchestrator, service map
and contract rules, with the children as gitignored clones and no CI — exists only when tasks
*routinely* cross repos. The teammate's instinct is right; what needs correcting is **what lives
in the workspace** (the orchestrator and the maps, not the specialists) and **how it delegates**.

```
Layer 3  workspace repo              ← .code-workspace, cross-repo orchestrator, service map, contracts, sync script
Layer 2  org-level agents            ← .github-private/agents/ (Business/Enterprise) — specialists every repo would otherwise copy
Layer 1  each repo (required)        ← copilot-instructions.md, .github/agents, instructions, skills, hooks — chapters 1-12
```

## What the platform does across several folders

This is the design constraint, and it is *different* from the Claude Code edition:

| Thing | VS Code multi-root workspace | Copilot CLI run in a child | Cloud agent |
|---|---|---|---|
| `copilot-instructions.md` / `AGENTS.md` | Discovered **per workspace folder** | The child's | The child's |
| `.github/instructions/` (`applyTo:`) | Per folder; globs match within that folder | The child's | The child's |
| `.github/agents/*.agent.md` | **Every folder's agents are loaded** — identity is the `name` field, so two repos' `backend-api` collide | Only the child's (plus `~/.copilot/agents/`) | The assigned repo's (plus org-level) |
| `.github/skills/` | Per folder | The child's | The child's |
| `.github/hooks/*.json` | Per folder (VS Code also reads `.claude/settings.json`) | The child's | Only `.github/hooks/*.json` of the assigned repo |
| Cross-repo changes in one run | Possible (it is one editor session) | No — run it per repo | **No** — "Copilot cannot make changes across multiple repositories in one run" |

Two consequences drive everything below:

1. **A multi-root session sees every child's enforcement layer** (instructions, hooks) — good —
   **and every child's agents** — which means *name collisions*. If three services each ship a
   `backend-api` agent, the workspace session has three agents called `backend-api` and which one
   answers is not something you control.
2. **The cloud agent is per repo, full stop.** A cross-repo change is N cloud-agent tasks, sequenced
   by the contract protocol, never one.

Also verified, and it corrects a v1.0 claim: **custom agents can invoke custom agents.** In VS Code
the `agents:` frontmatter list is an allowlist of subagents (`*` = all) and nested invocation is
enabled with `chat.subagents.allowInvocationsFromSubagents`; GitHub's `agent` tool alias does the
same on the cloud agent. A workspace orchestrator → repo orchestrator → repo specialists tree is
possible inside one VS Code session.

## Layer 1 — every repo keeps its own framework install

Nothing here removes the per-repo install. Each API repo, the web app and the iOS app keep
`.github/copilot-instructions.md`, `.github/agents/<repo>-orchestrator.agent.md` + specialists
(for the iOS repo: the `ios-*` roster from Chapter 3), `.github/instructions/`, `.github/skills/`,
`.github/hooks/` and `docs/ai-context/`. Reasons:

- Most tasks are single-repo. An iOS engineer opening the app repo — or a backend team opening
  theirs — must get the full experience with zero knowledge that a workspace exists.
- The cloud agent and the CLI see only the repo they run in — the gates must live with the code.
- The child's orchestrator is what the workspace delegates to.

**Naming rule for repos that will join a workspace:** the orchestrator is `<repo>-orchestrator`
(unique by construction — `ios-app-orchestrator`, `orders-api-orchestrator`). Specialists keep
generic names (`backend-api`; `ios-network`) *inside* the repo; the workspace never invokes them
directly (see mechanisms below), so collisions are avoided by design rather than by renaming every
specialist in every repo. Note that the iOS edition's own `ios-*` names already avoid colliding with
a backend repo's `backend-api` / `database` — but two iOS repos (an app and a shared SDK) would
collide on `ios-network`, which is why the rule is "invoke orchestrators only".

## Layer 2 — shared specialists live at the organisation level

Twelve backend services will share `backend-api`, `database`, `observability`, `release-devops`,
`security-privacy` almost verbatim; an iOS app and an iOS SDK repo will share `ios-tests`,
`ios-privacy` and `ios-release`. Copying them into every repo guarantees drift. Copilot's own
mechanism is **organisation-level custom agents** in the `/agents/` directory of the org's
`.github` or `.github-private` repository (Business / Enterprise; precedence by file name is
repo > org > enterprise, so a repo can still override one). Put the generic specialists and the
shared skills there; keep in each repo the router, the repo-specific instruction files, the
orchestrator, `PROJECT.md`/`LEARNINGS.md`/backlogs, and any specialist unique to the repo.

Personal-tier Copilot has no org level — keep copies, and make the quarterly REFINEMENT pass diff
them. Skip this layer below three repos.

## Layer 3 — the workspace repo

Create it when a task description regularly contains more than one repo name. Layout:

`templates/workspace/bootstrap.sh.template` creates all of this from a filled `workspace.json` (it fills the per-repo placeholders, generates `.gitignore` and the `.code-workspace` folder list from the manifest's paths, fills the orchestrator's `agents:` allowlist and the service-map rows, and lists what is left to fill by hand). The layout it produces:

```
workspace/                                  ← its own git repo; NO CI; deploys nothing
├── <ws>.code-workspace                      ← folders: ".", "ios-app", "services/orders-api", "web", "android-app", …
├── workspace.json                           ← manifest: repos, contracts, delegation defaults
├── .gitignore                               ← every child clone directory
├── .github/
│   ├── copilot-instructions.md              ← workspace router (templates/workspace/copilot-instructions.md.template)
│   ├── agents/
│   │   ├── <ws>-orchestrator.agent.md       ← cross-repo coordinator; agents: [contract-guardian, service-mapper]
│   │   ├── contract-guardian.agent.md       ← REVIEW-ONLY: contract diff + consumer grep across folders
│   │   └── service-mapper.agent.md          ← read-only: who owns what; maintains SERVICE_MAP.md
│   ├── instructions/
│   │   └── cross-repo-contracts.instructions.md   ← applyTo: "**" — the contract protocol, always on
│   ├── skills/
│   │   └── delegate/SKILL.md                ← /delegate <repo> <handoff> — runs the child's orchestrator via the CLI
│   └── hooks/framework.json                 ← OPTIONAL — not among the 14 workspace templates; copy templates/hooks/ and keep correction-capture only
├── scripts/
│   ├── sync-repos.sh                        ← clone/fetch every repo in the manifest; never resets a branch
│   └── delegate.sh                          ← cd <child> && copilot -p … --agent=<repo>-orchestrator
├── docs/ai-context/
│   ├── SERVICE_MAP.md · CONTRACTS.md · HANDOFF_SCHEMA.md
├── ios-app/  services/orders-api/  web/     ← gitignored clones
```

Plain clones (not submodules) driven by `workspace.json` + `sync-repos.sh`. Submodules pin SHAs,
which fights active development. The `.code-workspace` file is what the teammate meant by "a
workspace file" — it is the right artefact; it just should not be the only one.

**No CI at the workspace level.** Each repo deploys itself. The one optional job is a scheduled
`contract-guardian` run that opens an issue on drift.

### The workspace orchestrator's contract

A coordinator with no edit tools. Its job:

1. Restate the cross-repo task; read `SERVICE_MAP.md` and `CONTRACTS.md`; list the repos and the
   contracts between them.
2. **Order by contract** — producer first, additive only, deployed (check the producer's
   `PROJECT.md` §3, not its git log) before any consumer depends on it.
3. **One handoff per repo**, to that repo's `<repo>-orchestrator`, using the standard schema plus
   `repo:` and `contract_impact:`. Sequential for repos that share a contract.
4. Delegate with one of the mechanisms below; validate every `return:`; refuse to declare done while
   any `contract_impact.consumers_to_update` lacks a completed consumer handoff.

### Three delegation mechanisms — pick by surface and by whether the child is written to

**Mechanism A — the child's own Copilot CLI session** (default for writes, and the only one that
works from a script). `scripts/delegate.sh <repo> <handoff>` runs, *from inside the child
directory* (the CLI has no `--cwd`; it loads the working directory's customizations):

```bash
cd "<repo path>" && copilot -p "$(cat handoff.yaml)" --agent="<repo>-orchestrator" \
  --allow-tool="<TOOL_SPEC>" … -s
```

The child's instructions, agents, skills and `.github/hooks/*.json` all load — its build-gate and
doc-gate fire. Only the child's agents exist in that process, so **no name collisions**. The
`return:` block comes back in the text output (a documented structured-output flag is not available
on the CLI as of the verification date — the script extracts the YAML block). Pre-approve the tools
the child needs with `--allow-tool=`; never `--allow-all-tools` on a repo you don't fully trust —
`-p` runs that repo's hooks without a prompt.

**Mechanism B — in-editor subagents** (interactive cross-repo work in VS Code). The workspace
orchestrator's `agents:` lists the child orchestrators by name (`ios-app-orchestrator`,
`orders-api-orchestrator`, …); VS Code loads them from the child folders; `runSubagent` invokes them;
with `chat.subagents.allowInvocationsFromSubagents` enabled each child orchestrator can in turn
invoke its own specialists. Every child's `applyTo:` instructions and hooks apply because they are
workspace folders. The cost is the collision hazard from the table above: **only orchestrators are
invoked by name from the workspace level** (unique by construction), and if two children ship
same-named specialists, the child orchestrators' own `agents:` lists cannot disambiguate them —
prefer Mechanism A for writes in that case.

**Mechanism C — the cloud agent, one task per repo.** The workspace orchestrator (or a human) files
one issue per repo, in contract order, each carrying the handoff block in the issue body and
assigned to that repo's orchestrator agent. The cloud agent loads that repo's `.github/hooks/*.json`
and agents. There is no cross-repo run; sequencing is the protocol's job.

### Two new handoff fields (additive — `schema_version` stays 1)

Outbound `repo:` and `contract_impact: {level, contracts, consumers_to_update}`; inbound
`contracts_changed: [{contract, change, backward_compatible, consumers_grepped}]`. Full definitions
in `docs/04-HANDOFF-SCHEMA.md`. A specialist that finds itself needing a second repo returns
`status: blocked` with `recommended_next_agent` naming that repo's orchestrator — it never reaches
across, and on the cloud agent it physically cannot.

### The cross-repo contract protocol (`.github/instructions/cross-repo-contracts.instructions.md`)

The multi-client parity rules from Chapter 12, across repository boundaries:

1. **Producer first, additive only, deployed before any consumer depends on it.** A breaking change
   is add → move every consumer → remove; never one handoff.
2. **Grep every consumer in the same turn** and list them in `consumers_grepped` — including the
   zero-hit ones. Clients pin shapes; a rename fails silently at runtime.
3. **Never ship a consumer that hard-depends on a producer change that might not be deployed.**
   Probe or degrade, and say so on screen when degraded.
4. **Same heading ⇒ same endpoint.** Before building a panel that mirrors another consumer's,
   open that consumer's code and find what it fetches; record the pairing in `CONTRACTS.md`.
5. **One brain.** Logic in the producer; consumers render.
6. **One session for a whole cross-repo change.** Plan first, then delegate in contract order.

### `workspace.json` and the `.code-workspace`

The manifest (`templates/workspace/workspace.json.template`) lists each repo's `path`, `url`,
`default_branch`, `orchestrator`, `build`, `test`, `owns`, plus every contract (producer,
consumers, spec path, versioning rule) and the delegation defaults. The `.code-workspace` file
(`templates/workspace/workspace.code-workspace.template`) lists the same paths as folders, with
the workspace root first, and sets `chat.subagents.allowInvocationsFromSubagents` for Mechanism B.

### Shared Swift packages across repos

A shared SPM package repo (a networking kit, a design system, a generated API client) is itself a
**producer with consumers** — every app and every SDK that declares it in `Package.swift` pins its
public API the way a client pins a JSON shape. Treat it exactly as a contract in `CONTRACTS.md`:
`kind: shared-schema`, the package repo as producer, each app/SDK repo as a consumer, and the
versioning rule written down. **Pin by version tag, never by branch**, in every consumer's
`Package.swift` (`.package(url: …, exact: "2.3.0")` or a `from:` range you actually mean — never
`branch: "main"`): a branch pin turns every push to the package into an unreviewed, unversioned
change to every consumer's next build, and the producer-first / additive-only / deployed-before-
consumers ordering below has nothing to order against. A breaking change to the package is a tag
bump plus one consumer handoff per app, in contract order, like any other contract.

## Scheduled workspace routines — the contract guardian gets a clock

Agent-era defects concentrate at the seams between repos — and the iOS app is a consumer of every
contract, so drift lands here first. Seams are exactly where standing routines (Chapter 14) pay
off:

- **contract-drift-daily** — re-derive each contract's consumer list from the child clones and diff
  reality against `CONTRACTS.md` + `SERVICE_MAP.md`; include shared Swift packages in the diff (a
  package pinned by branch instead of tag is drift in waiting). Open a workspace PR (or per-repo
  issues) on divergence — the `contract-guardian` charter on a schedule, same REVIEW-ONLY posture.
- **service-map-freshener** — re-verify `SERVICE_MAP.md` rows against the clones; a stale row gets
  a dated PR, not silent tolerance (Chapter 12's freshness contract, applied cross-repo).
- **workspace janitor** — prune stale clones and re-sync from the manifest (`sync-repos.sh`), so
  delegation never runs against a repo state nobody chose.

Routine writes follow the same delegation rule as everything else in this chapter: a routine that
must *change* a child repo delegates to that child's own orchestrator session; the workspace-level
routine itself stays read-only plus reports. Conventions, budgets and the catalog:
`docs/14-STANDING-ROUTINES.md`.

## What NOT to do

- **Don't put the specialists in the workspace repo.** They belong with the code they edit, or at
  the org level.
- **Don't invoke a child's *specialist* by name from the workspace session.** Names collide across
  folders. Invoke the child's orchestrator and let it route.
- **Don't expect one cloud-agent task to touch two repos.** It cannot. One task per repo, in
  contract order.
- **Don't use git submodules for the children.** Manifest + gitignored clones.
- **Don't give the workspace a deploy pipeline.**
- **Don't run `copilot -p` with `--allow-all-tools` against a repo you don't fully trust** — it
  runs that repo's hooks unprompted.
- **Don't fan out writes to a producer and its consumer in parallel.**
- **Don't carry a production approval across repos.** Per repo, per incident, current turn.

## POC recipe (one afternoon)

1. Pick one API repo and the iOS app, which share a contract (the orders API and the order screen,
   say). Confirm each has a layer-1 install with a working `<repo>-orchestrator` (open the repo in
   VS Code, select the orchestrator, give it a single-repo bug — for the iOS repo, one that needs
   `xcodebuild` in `tests_run`). If not, do that first.
2. Create the workspace repo from `templates/workspace/`. Fill `workspace.json` and the
   `.code-workspace` with the two repos and the one contract. Run `scripts/sync-repos.sh`. Open the
   `.code-workspace` in VS Code.
3. Select `<ws>-orchestrator` and ask for something *additive* that crosses the contract ("add
   `estimatedDelivery` to the orders API response and show it on the iOS order screen").
4. Verify, in order: it read `CONTRACTS.md` and ordered producer → consumer; the producer handoff
   carried `repo:` and `contract_impact.level: additive`; the producer's own instruction files and
   hooks fired (Mechanism A: in the CLI output; Mechanism B: in the child folder's hook behaviour);
   the iOS return carried `consumers_grepped` and an `xcodebuild` line in `tests_run`, and the
   screen degrades (shows nothing, not a crash) when the field is absent — the installed app may
   reach users before the API deploys; `git status` in the workspace root is clean.
5. Break it on purpose: ask for a *rename*. The correct outcome is a refusal until the task is
   re-issued as add-then-remove with every consumer listed (the iOS app, and the web and Android
   clients if they read the same field).
6. Only then decide on layer 2: count identical agent files across the two repos.

## Cross-links

- `docs/02-ARCHITECTURE.md` § Variations — points here for multiple repositories.
- `docs/04-HANDOFF-SCHEMA.md` — the base schema and the two cross-repo fields.
- `docs/12-PROJECT-TRUTH-AND-LEARNINGS.md` § Multi-client parity — origin of the contract protocol.
- `docs/10-MECHANICAL-ENFORCEMENT.md` — the hooks each child brings with it.
- `templates/workspace/` — every file in the layout above.
- Official (verified 2026-08-22): *Custom agents configuration*, *VS Code → Custom agents*,
  *VS Code → Subagents*, *Agent customization overview* (per-folder discovery), *About Copilot
  coding agent* (single-repo), *Copilot CLI programmatic reference*, *Hooks reference*.
- `docs/14-STANDING-ROUTINES.md` — the routine conventions the scheduled layer above relies on.
