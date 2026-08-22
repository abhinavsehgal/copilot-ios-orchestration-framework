# 00 — Quickstart: onboard any iOS project (VS Code + Copilot CLI + Xcode, many repos)

> The whole framework as one step-by-step walk. Every step says **what to do**, **why**, **what to
> paste**, and **how you know it worked**. Written for an iOS engineer who has never seen the framework;
> the long version of each part is linked. Based on v1.1.0 and the GitHub Copilot + VS Code docs as
> verified on 2026-08-22 — if a step here disagrees with the chapter it links to, the chapter wins.
>
> **Time:** 15 min once per person · 2–4 h per repo · one afternoon for the workspace.

---

## Part 0 · The words (read once)

Nine words. Everything else in this guide is made of these.

| Word | Means |
|---|---|
| **Orchestrator** | The manager. Reads your task, decides who should do it, writes them a note, checks their work. **Never touches code itself.** |
| **Specialist** | A worker who does one kind of job (the API, the database, the UI, testing…). Touches only its own corner. |
| **Handoff** | The note the manager writes the worker: what to do, what the manager already knows (with proof), what would prove the manager wrong, what not to touch. |
| **Return** | The note the worker writes back: what it checked, what it changed, what it couldn't verify, what the manager must not pass on as fact. |
| **Instruction file** | A sticky note on a drawer. When anyone opens files in that drawer, the note appears. Lives in `.github/instructions/`. |
| **Skill** | A recipe card for a job you do again and again (`/commit-push-pr`, `/verify-build`). Lives in `.github/skills/`. |
| **Hook** | A tripwire. Runs a script when something happens (before a tool runs, when the agent tries to stop). Optional — add later. |
| **Workspace** | One desk with every repo's folder on it, plus one manager who only coordinates between repos. Its own small git repo. |
| **Contract** | The promise between a service and everyone who calls it (an API shape, an event payload, a shared package). The thing that breaks when two repos change at different times. |

---

## Part 1 · Before you start (15 minutes, once per person)

### 1. Make sure you have the five tools

**Why:** Xcode builds the app; VS Code runs the agents (Copilot's custom agents, skills and hooks
live there, not in Xcode); the Copilot CLI runs them from a terminal and is what the workspace uses
to reach into other repos; git and `jq` are used by the workspace scripts.

```bash
# Each of these should print a version, not an error
xcodebuild -version
code --version
copilot --version      # the GitHub Copilot CLI (the agentic one, not `gh copilot`)
git --version
jq --version
node --version         # only needed if you later turn on hooks
```

Missing `copilot`? Install it from GitHub's Copilot CLI page (docs.github.com → Copilot → Copilot
CLI → Install). Missing `jq`? `brew install jq` on a Mac, `winget install jqlang.jq` on Windows.

✓ **You know it worked when:** all six print a version, and in VS Code the Copilot Chat panel opens
with **Agent** in the mode dropdown. (Keep Xcode for building, running and Instruments; the agents
are driven from VS Code or the CLI.)

### 2. Get the framework onto your disk

**Why:** the framework is a folder of templates and two copy-paste prompts. You never install it
into your project — you point Copilot at it.

```bash
mkdir -p ~/frameworks
git clone https://github.com/abhinavsehgal/copilot-ios-orchestration-framework.git ~/frameworks/copilot-ios
echo ~/frameworks/copilot-ios     # remember this path — you will paste it into the prompts
```

✓ `ls ~/frameworks/copilot-ios` shows `docs  prompts  templates`.

### 3. Read one file

**Why:** twenty minutes now saves a confused afternoon later. Skip the rest of the docs for now.

Open `docs/09-IOS-RUNBOOK.md`. That is the long version of Part 2 below.

✓ You can say in one sentence what the orchestrator does and does not do. (It coordinates. It never edits.)

---

## Part 2 · One repo (2–4 hours — repeat for every repo)

Every repo gets its own manager and its own workers — the iOS app repo, and every API repo it
calls. The workspace in Part 3 only works if each repo already has this. Start with the iOS app.
The iOS roster is pre-filled (`ios-ui`, `ios-data`, `ios-network`, `ios-tests`, `ios-release`,
`ios-privacy`, `ios-perf`, `ios-bg`) — drop the ones your app doesn't need.

### 1. Open the repo in VS Code on a fresh branch

**Why:** the setup creates new files. A clean branch means you can throw it all away if you don't like it.

```bash
cd ~/code/<your-repo>
git checkout main && git pull
git checkout -b setup/copilot-orchestration
code .
```

✓ `git status` is clean and the bottom-left of VS Code says `setup/copilot-orchestration`.

### 2. Run the INVENTORY prompt (look, don't touch)

**Why:** Copilot scans the repo and *proposes* which specialists this repo needs. It writes nothing.
You correct the proposal before anything is created.

Open Copilot Chat → set the mode dropdown to **Agent**. Open
`~/frameworks/copilot-ios/prompts/INVENTORY-PROMPT.md`, copy all of it, paste it into the chat. Replace
`<framework path>` with the path from Part 1 step 2. Send.

It comes back with: the iOS specialists it thinks your app needs (usually 4–6 of the 8), proposed
instruction files with their globs (`Sources/**/Views/**/*.swift` → `swiftui.instructions.md`,
`*.entitlements` + `Info.plist` → `info-plist.instructions.md`, …), and a list of **Open Questions**. Answer the questions. Cross out specialists
that feel wrong. Fewer is better.

> ⚠ **The orchestrator must be named `<repo-name>-orchestrator`** (for example `orders-orchestrator`).
> Not just `orchestrator`. In Part 3 every repo's manager sits on the same desk, and names must not collide.

✓ You have a short list of specialist names you agree with, and `git status` is still clean.

### 3. Run the BOOTSTRAP prompt (now it builds)

**Why:** this creates all the files. It first **backs up** anything already in `.github/` and asks
before overwriting — so an existing `copilot-instructions.md` your team wrote is safe.

In the **same chat**, paste `~/frameworks/copilot-ios/prompts/BOOTSTRAP-PROMPT.md` (path replaced
again). It shows you each file before saving. Say yes to each one you agree with.

What it creates:

| File | What it is |
|---|---|
| `.github/copilot-instructions.md` | The router. Short. Read by every Copilot request. |
| `.github/agents/<repo>-orchestrator.agent.md` | The manager. |
| `.github/agents/ios-ui.agent.md`, `ios-data.agent.md`, … | The workers (the iOS roster you kept). |
| `.github/instructions/swiftui.instructions.md`, `concurrency…`, `coredata…`, `networking…`, `info-plist…`, `code-signing…` | The sticky notes, each with an `applyTo:` glob. |
| `.github/skills/<repo>-engineering/SKILL.md` | The six-gate working method every task follows. |
| `docs/ai-context/PROJECT.md` | "What is true right now" — what's live where, verified commands. |
| `docs/ai-context/LEARNINGS.md` | Decisions, failed approaches, corrections. |
| `docs/ai-context/HANDOFF_SCHEMA.md`, `INDEX.md`, `GLOSSARY.md` | The note format, the map, the one-name-per-thing list. |
| `docs/<AREA>_BACKLOG.md` | Where "we'll do it later" must be written down. |

✓ `ls .github/agents` lists your orchestrator and specialists. The app still builds:
`xcodebuild -workspace <App>.xcworkspace -scheme <App> -destination 'generic/platform=iOS Simulator' build`.

### 4. Check the manager shows up

**Why:** an agent file with a typo in its name silently doesn't load.

In VS Code: **Developer: Reload Window** (Ctrl/Cmd+Shift+P). Open Copilot Chat. Click the agent
dropdown (next to the mode dropdown).

✓ You see `<repo>-orchestrator` and every specialist in the dropdown.

> ⚠ Not there? The file must end in `.agent.md`, the `name:` line must match, and the YAML between
> the `---` lines must be valid. Fix, reload, look again.

### 5. Give it one real, small job

**Why:** the only proof the setup works is watching a handoff go out and a return come back.

Select `<repo>-orchestrator` in the dropdown. Type a real bug or tiny feature from your tracker —
one that needs maybe 30 minutes of human work (a missing accessibility label, a `@StateObject` that
should be `@ObservedObject`). Send.

Watch for three things: it restates the task; it writes a `handoff:` block (YAML) with `claims`,
`failure_condition`, `in_scope`; the specialist answers with a `return:` block that lists
`files_changed` and `tests_run`.

✓ You saw both blocks. The change is small and correct. If the specialist *refused* a vague
handoff — that's the framework working, not failing.

### 6. Commit, open a PR, merge

**Why:** teammates get the agents the moment they pull. The backup folder stays local.

```bash
git add .github/ docs/ .gitignore
git commit -m "chore: bootstrap Copilot orchestration framework"
git push -u origin setup/copilot-orchestration
gh pr create --base main --title "Bootstrap Copilot orchestration"
```

✓ PR merged. A teammate pulls, reloads VS Code, and sees the same agents in their dropdown.

### 7. Later, not now: turn on hooks

**Why:** hooks are tripwires (block a stop while `xcodebuild` is red; force a correction to become
an instruction file; refuse to end a turn after a production push with stale docs). Add them only
after you've seen people skip the written rules. `docs/10-MECHANICAL-ENFORCEMENT.md` is the recipe;
the build-gate template is already filled with `xcodebuild` and a 20-minute internal cap.

```bash
# when you're ready:
cp ~/frameworks/copilot-ios/templates/hooks/hooks.json.template .github/hooks/framework.json
mkdir -p .github/scripts && cp ~/frameworks/copilot-ios/templates/hooks/*.mjs.template .github/scripts/
# rename *.mjs.template → *.mjs, fill the <PLACEHOLDERS>, restart VS Code
```

> ⚠ Hooks are read when a session starts. After installing: reload VS Code / start a new `copilot`
> session, then test. Hook timeouts **fail open** — a slow `xcodebuild` silently allows, which is
> why the gate keeps its own cap *below* the hook timeout and treats a killed build as inconclusive.

**Now do Part 2 again for the next repo.** Two repos done with working orchestrators is the minimum
before Part 3.

---

## Part 3 · The workspace (one afternoon, once per team)

Only when tasks keep saying "change the orders API *and* the app". The usual iOS shape is one app
repo that *consumes* several API repos (and maybe a shared Swift package, which is a contract too —
pin it by version tag, never by branch). The workspace is a small
extra repo that holds one coordinating manager, a map of who owns what, and the rules for changing
contracts. It holds **no workers** — they stay in their repos. Long version:
`docs/13-MULTI-REPO-WORKSPACES.md`.

### 1. Create an empty repo and clone it

**Why:** the workspace is its own repo so the map and the rules are versioned and shared. It has no
CI and deploys nothing.

```bash
gh repo create <team>-workspace --private --clone
cd <team>-workspace
```

✓ You're inside an empty folder with a `.git`.

### 2. Copy the workspace templates into place

**Why:** fourteen files, each with a fixed home. `templates/workspace/README.md` has the same table.

```bash
T=~/frameworks/copilot-ios/templates/workspace
mkdir -p .github/agents .github/instructions .github/skills/delegate scripts docs/ai-context
cp $T/copilot-instructions.md.template              .github/copilot-instructions.md
cp $T/workspace.code-workspace.template             <team>.code-workspace
cp $T/workspace.json.template                       workspace.json
cp $T/gitignore.template                            .gitignore
cp $T/orchestrator-agent.md.template                .github/agents/<team>-orchestrator.agent.md
cp $T/contract-guardian-agent.md.template           .github/agents/contract-guardian.agent.md
cp $T/service-mapper-agent.md.template              .github/agents/service-mapper.agent.md
cp $T/cross-repo-contracts.instructions.md.template .github/instructions/cross-repo-contracts.instructions.md
cp $T/delegate-skill/SKILL.md.template              .github/skills/delegate/SKILL.md
cp $T/sync-repos.sh.template scripts/sync-repos.sh
cp $T/delegate.sh.template   scripts/delegate.sh
cp $T/SERVICE_MAP.md.template docs/ai-context/SERVICE_MAP.md
cp $T/CONTRACTS.md.template   docs/ai-context/CONTRACTS.md
cp ~/frameworks/copilot-ios/templates/HANDOFF_SCHEMA.md.template docs/ai-context/HANDOFF_SCHEMA.md
chmod +x scripts/*.sh
```

✓ `find . -type f -not -path './.git/*'` matches the layout in chapter 13.

### 3. Fill in the blanks — mostly in one file

**Why:** `workspace.json` is the source of truth: which repos, where they live, what each one's
manager is called, and every contract between them. The scripts read it; you never hard-code a path
anywhere else.

Open `workspace.json`. For each repo fill `name`, `path` (e.g. `ios-app`, `services/orders-api`),
`url`, `default_branch`, `orchestrator` (exactly the name from Part 2 — `ios-app-orchestrator`,
`orders-api-orchestrator`), `build` (`xcodebuild … build` for the app), `test`. For each contract fill `producer`, `consumers`, `spec` (where the API/event spec file
lives), and a one-line `versioning` rule.

Then search every copied file for `<` and replace the remaining placeholders (`<WORKSPACE_SLUG>`,
`<REPO_1_NAME>`…). The `.code-workspace` folder list must match the `path`s in `workspace.json`.

✓ `grep -rn "<[A-Z_0-9]*>" . --exclude-dir=.git` prints nothing. `jq . workspace.json` prints valid JSON.

### 4. Pull every repo onto the desk

**Why:** the children are plain clones in gitignored folders — disposable. The script clones what's
missing and fetches what exists. It **never** switches or resets a branch; each repo's team owns that.

```bash
scripts/sync-repos.sh
```

✓ One `==` line per repo, no `⚠` warnings. A warning means that repo has no `<repo>-orchestrator`
yet — go back to Part 2 for it.

### 5. Open the desk in VS Code

**Why:** VS Code reads each folder's `.github/` separately, so every repo's agents, sticky notes
and skills are all available at once. The workspace file also switches on nested subagents.

**File → Open Workspace from File…** → pick `<team>.code-workspace`. Reload the window once it opens.

✓ The Explorer shows the workspace folder plus every child. The agent dropdown shows
`<team>-orchestrator`, `contract-guardian`, `service-mapper` *and* each repo's own orchestrator.

> ⚠ Every repo ships a worker called `backend-api`. On the desk those names collide. Rule: from the
> workspace, **only ever pick an orchestrator by name**. Never a worker. The workspace manager
> already knows this.

### 6. Commit the workspace

```bash
git add -A
git commit -m "chore: bootstrap cross-repo workspace"
git push -u origin main
```

✓ `git status` shows no child repo files — the clones are ignored, the map and rules are committed.

### 7. The first cross-repo job — add something

**Why:** the workspace's whole job is ordering changes across a contract: the API that *produces* a
field changes first, is deployed, and only then does the app that *reads* it change — an installed
build can be older *or newer* than the server, so the app must probe or degrade, never assume.

Select `<team>-orchestrator`. Ask for something additive that crosses one contract, for example:

```text
Add an `estimatedDelivery` field to the orders API response and show it on the iOS order detail screen.
```

Watch for: it reads `CONTRACTS.md`; it says "producer first: orders-api, then ios-app"; it writes
one handoff per repo, each with `repo:` and `contract_impact: additive`; it hands the API job to
`orders-api-orchestrator` (in chat) or runs `scripts/delegate.sh orders-api …` (from a terminal —
that runs the API repo's own Copilot session so its own rules and hooks apply); the app handoff
says the new field is optional and the screen degrades when it is absent.

✓ Two handoffs, in that order. Each return lists `consumers_grepped`. `git status` in the workspace
root is clean — the workspace itself changed nothing.

### 8. Break it on purpose — rename something

**Why:** a rename is a breaking change. The correct answer is a refusal.

```text
Rename `total` to `grandTotal` in the orders API response.
```

✓ It refuses to run it as one change and proposes three: add `grandTotal` → move every consumer
(the app, and any other client) → remove `total` after every consumer has shipped. If it just did the rename, the contract rule is not loading — check
`.github/instructions/cross-repo-contracts.instructions.md` has `applyTo: "**"`.

---

## Part 4 · Every day (the whole thing in one table)

| You want to… | Where | Do this |
|---|---|---|
| Fix a typo, ask a question | That repo, VS Code | Plain chat, no agent selected. The framework stays out of your way. |
| Do a medium/complex job inside one repo | That repo, VS Code | Pick `<repo>-orchestrator` in the dropdown. Describe the job. Read the handoff and the return. |
| Same, from a terminal | That repo, CLI | `cd` into the repo, run `copilot`, use `--agent=<repo>-orchestrator` or pick it inside. Same agents, same rules. |
| Ship what you did | Any | `/commit-push-pr` — it refuses to push to `main`, builds first, never stages secrets. |
| Change something in the API *and* the app | The workspace (`.code-workspace`) | Pick `<team>-orchestrator`. It orders the work producer → consumer and delegates one repo at a time. |
| Ask "is this API change safe?" | The workspace | Pick `contract-guardian`. It greps every consumer and answers with file:line, never an opinion. |
| Correct the agent ("no, use X") | Any | Then type `/correction-capture`. It turns the correction into a sticky note so it never happens again. Never accept "I'll remember". |
| Say "we'll do that later" | Any | It must be written into `docs/<AREA>_BACKLOG.md` before the chat ends. Spoken follow-ups vanish. |
| Submit to App Review / promote a TestFlight build | The app repo | Same session: update `CHANGELOG.md`, the affected `docs/ai-context/` map, and `PROJECT.md` "what is live where" (App Store / TestFlight / internal). A fresh agent sees only what's in git. |
| Hand a well-defined task to the cloud agent | GitHub issue | One issue per repo, assigned to that repo's orchestrator agent. It cannot touch two repos in one run. |

---

## Part 5 · When it goes wrong

| You see | It usually means | Fix |
|---|---|---|
| The agent isn't in the dropdown | File name or frontmatter | Must end in `.agent.md`; `name:` must match; valid YAML between the `---` lines. Reload Window. |
| A sticky note never shows up | Its `applyTo:` glob doesn't match | Run `find . -path "<the glob>"` — if nothing prints, the glob is wrong. Globs are relative to that repo's root. |
| The wrong repo's worker answered | Name collision on the desk | From the workspace, invoke orchestrators only. The workspace router says so in rule 4. |
| `delegate.sh` exits with "handoff does not carry repo:" | The handoff file is missing its `repo:` line | Add `repo: <name>` exactly as in `workspace.json`. A note for one repo must never run in another. |
| `delegate.sh`: command not found / no such repo | `copilot` not on PATH, or a bad `path` in the manifest | `copilot --version`; then `jq '.repos[].path' workspace.json` and check each folder exists. |
| A hook never fires | Installed mid-session, or in the wrong place | Restart the session. Team hooks live in `.github/hooks/*.json` (committed); `~/.copilot/hooks` is only you. Timeouts fail open — the `xcodebuild` gate's own cap must sit below `timeoutSec`. |
| The build-gate blocks on a "failed" build that was just slow | Derived-data contention or a low cap | A build killed by the cap must *allow* (the template does). Raise `BUILD_CAP_MS`; don't run two `xcodebuild`s in one derived-data folder. |
| The specialist refused the task | The handoff was vague | That's correct behaviour. Add the missing field it named (usually `failure_condition` or evidence on a claim) and re-send. |
| The manager is editing code itself | Specialists too narrow, or the wrong agent selected | Check the dropdown. If it really is the orchestrator, widen a specialist's scope rather than letting the manager code. |
| Something worked last month and now doesn't | The platform moved | Re-run `prompts/REFINEMENT-PROMPT.md`; its "platform drift" section re-checks the Copilot docs. Three framework claims went stale in three months once already (Pitfall 23). |

---

## The checklist (print this)

**Per person, once**
- [ ] `xcodebuild`, `code`, `copilot`, `git`, `jq` all print a version
- [ ] Framework cloned to `~/frameworks/copilot-ios`

**Per repo**
- [ ] Branch `setup/copilot-orchestration`
- [ ] INVENTORY prompt → specialist list agreed
- [ ] BOOTSTRAP prompt → files created; orchestrator named `<repo>-orchestrator`; the app still builds with `xcodebuild`
- [ ] Reload Window → agents visible in the dropdown
- [ ] One real small job → saw `handoff:` and `return:`
- [ ] PR merged; a teammate sees the agents too
- [ ] (later) hooks installed, session restarted, tested

**Per team, once — after ≥ 2 repos are done**
- [ ] Workspace repo created; 14 template files in place; no `<PLACEHOLDER>` left
- [ ] `workspace.json`: every repo + every contract filled
- [ ] `scripts/sync-repos.sh` → no warnings
- [ ] Opened the `.code-workspace`; workspace orchestrator + each repo's orchestrator visible
- [ ] Additive cross-repo job → producer first, two handoffs, consumers grepped
- [ ] Rename job → refused and split into add / move / remove

---

## Cross-links

- `docs/00-QUICKSTART.html` — this guide plus the other two editions as one offline page with tabs (open in a browser; generated from the three `00-QUICKSTART.md` files on 2026-08-22).

- `docs/09-IOS-RUNBOOK.md` — the long version of Part 2.
- `docs/10-MECHANICAL-ENFORCEMENT.md` — hooks (Part 2 step 7).
- `docs/13-MULTI-REPO-WORKSPACES.md` — the long version of Part 3 (one iOS app + N API repos; shared Swift packages as contracts).
- `docs/12-PROJECT-TRUTH-AND-LEARNINGS.md` — why the backlog, `PROJECT.md` and "submit = freshen docs" rules exist.
- `docs/08-IOS-COMMON-PITFALLS.md` — Part 5 in full, plus the nine iOS platform pitfalls (32–40).
