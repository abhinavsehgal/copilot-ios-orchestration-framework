# BOOTSTRAP-PROMPT (iOS-anchored)

> **Paste this entire file into Copilot Chat AFTER you've run INVENTORY-PROMPT and answered its questions. This prompt generates real files in `.github/`, `docs/ai-context/`, and `docs/_archive/`. Pre-flight safety checks run BEFORE any file is written.**

You are running the bootstrap pass for the [Copilot iOS Orchestration Framework](https://github.com/abhinavsehgal/copilot-ios-orchestration-framework). Your job is to **generate the framework files** for this iOS project, using the answers from the previous INVENTORY pass and the templates at `<framework path>/templates/`.

## Phase A — Mandatory pre-flight (before ANY file is written)

### Pre-flight 1 — Auto-snapshot

If `.github/` exists with any framework-managed files (any of: `copilot-instructions.md`, `instructions/`, `prompts/`, `agents/`, `chatmodes/`):

```bash
mkdir -p .github-pre-bootstrap-backup
cp -r .github/copilot-instructions.md .github-pre-bootstrap-backup/ 2>/dev/null
cp -r .github/instructions .github-pre-bootstrap-backup/ 2>/dev/null
cp -r .github/prompts .github-pre-bootstrap-backup/ 2>/dev/null
cp -r .github/agents .github-pre-bootstrap-backup/ 2>/dev/null
cp -r .github/chatmodes .github-pre-bootstrap-backup/ 2>/dev/null
```

Add `.github-pre-bootstrap-backup/` to `.gitignore` if not already present.

Report what was backed up.

### Pre-flight 2 — Naming-collision check

For each file the framework would generate (per the INVENTORY proposal):
- If the file path already exists with non-trivial content (>200 chars), STOP and emit `<NEEDS USER CONFIRMATION>` flag. Show:
  - Existing file's first 30 lines
  - Proposed new file's first 30 lines
  - Three options: (a) overwrite, (b) keep existing untouched, (c) merge (you'll guide the user line-by-line)

Do not proceed past pre-flight 2 until every collision is explicitly resolved.

### Pre-flight 3 — `applyTo:` glob conflict check

For each proposed instruction file's `applyTo:`, check if any existing `.github/instructions/*.instructions.md` declares an overlapping glob. If yes:
- Show both globs side by side
- Show real files that match both
- STOP with `<NEEDS USER CONFIRMATION>` — overlapping globs cause contradictory rule loads

### Pre-flight 4 — Drift detection on existing `copilot-instructions.md`

If existing `.github/copilot-instructions.md` is present:
- Read it fully
- Identify "router-shaped" content vs "orchestrator-persona-shaped" content
- If the existing file has orchestrator-persona content (delegation instructions, agent personalities, multi-step workflows): STOP with `<NEEDS USER CONFIRMATION>` — the framework will move that to `.github/agents/<your-app>-orchestrator.md` and slim the router; user must approve

### Pre-flight 5 — Existing agent / chatmode style detection

If `.github/agents/` or `.github/chatmodes/` already has content:
- List existing agent names + 1-line descriptions
- Compare against proposed iOS specialist roster
- Flag conflicts (e.g. existing `database` agent vs proposed `ios-data` — should they merge or coexist?)
- STOP with `<NEEDS USER CONFIRMATION>` per conflict

### Decision gate

After pre-flights 1-5, summarize:
- Files to be created (clean — no conflicts)
- Files needing user decision (with each conflict's resolution path)
- Files to skip (user said keep-existing during pre-flight)

User must explicitly say "proceed" or "abort" before Phase B.

## Phase B — File generation

For each file in the approved list (from Phase A's decision gate):

1. **Read the corresponding template** from `<framework path>/templates/`
2. **Substitute placeholders** with values from INVENTORY answers:
   - `<PROJECT_NAME>` → user's confirmed project name
   - `<PROJECT_SLUG>` → kebab-case slug
   - `<DEFAULT_BASE_BRANCH>` → user's confirmed branch (default `develop`)
   - `<BUILD_COMMAND>` → user's confirmed build command (default `xcodebuild -workspace MyApp.xcworkspace -scheme MyApp build`)
   - `<TEST_COMMAND>` → user's confirmed test command
   - `<IOS_DEPLOYMENT_TARGET>` → user's confirmed iOS minimum (e.g. `15.0`)
   - `<XCODE_WORKSPACE>` → workspace filename
   - `<XCODE_SCHEME>` → primary scheme
   - `<PROJECT_TRAILER>` → team's standard commit trailer
   - `<APPLY_TO_GLOB_*>` → globs proposed in INVENTORY (with user's confirmed adjustments)
3. **Show preview** — display the final file content (first 50 lines + length) before writing
4. **Wait for user confirmation** ("yes" / "skip" / "edit") per file. Default: "yes" if the file is in the no-conflicts list AND user said "proceed all" at the decision gate.
5. **Write the file**

### Files to generate (in this order — dependencies between later files reference earlier ones)

1. `.gitignore` (only if not present; otherwise append the framework's required entries: `.github-pre-bootstrap-backup/`, `*.cer`, `*.p12`, `*.mobileprovision`, `Profiles/*`)
2. `docs/_archive/README.md` (frozen archive marker)
3. `docs/ai-context/INDEX.md` (the router orientation map)
4. `docs/ai-context/HANDOFF_SCHEMA.md` (the bidirectional contract)
5. `docs/ai-context/ORCHESTRATION_SPOONFEEDER.md` (human-readable usage guide)
6. `.github/copilot-instructions.md` (thin router — golden rules + workflow + cross-links)
7. `.github/agents/<project-slug>-orchestrator.md` (orchestrator persona)
8. Each iOS specialist agent file (in user's confirmed list; defaults to all 8)
9. Each `.github/instructions/*.instructions.md` (in user's confirmed list)
10. `.github/prompts/correction-capture.prompt.md` (iOS-flavored)
11. `.github/prompts/commit-push-pr.prompt.md` (iOS-pre-filled with build/test commands)
12. `.github/prompts/verify-build.prompt.md` (iOS-pre-filled)
13. (Optional) Tier-2 canonical doc stubs: `docs/ARCHITECTURE.md`, `docs/BUILD_SYSTEM.md`, `docs/PRIVACY_AND_DATA.md`, etc. — only if user said yes in INVENTORY

Each generated file gets:
- A clean header with file purpose
- Placeholders substituted
- Cross-links updated to point to actual generated paths
- The universal evidence rule preamble (for instruction files and agent files)

## Phase C — Post-generation verification

After all files are written:

1. List all generated paths in a single block (so the user can `git add` them in one command)
2. Run a quick sanity check: read each generated file's first line; confirm no template placeholders survived (no `<UPPERCASE>` strings in the output)
3. Suggest the next 3 actions:
   ```bash
   # 1. Sanity-check that the build still passes
   <BUILD_COMMAND>

   # 2. Add and commit
   git add .github/ docs/ai-context/ docs/_archive/ .gitignore
   git commit -m "chore: bootstrap Copilot iOS orchestration framework"

   # 3. Push and PR
   git push -u origin setup/copilot-ios-framework
   gh pr create --base <DEFAULT_BASE_BRANCH> --title "Bootstrap Copilot iOS orchestration framework"
   ```

4. Suggest **Phase 4** of the runbook (first task end-to-end via the orchestrator) — paste the orchestrator's name + a small real task to validate the framework works.

## Hard NOs in Phase B

- **NEVER** write any file under `Sources/`, `Tests/`, `Pods/`, `*.xcodeproj/`, `*.xcworkspace/` — those are the user's app code, not framework files
- **NEVER** write to `.cer`, `.p12`, `.mobileprovision`, `*.keychain-db`, or any path containing `secret` / `credentials` / `private-key`
- **NEVER** stage / commit / push as part of bootstrap — that's the user's call after they review the generated files
- **NEVER** skip pre-flight 1 (the snapshot) — even on a "clean" repo, the safety net is cheap
- **NEVER** write a file that has unresolved `<UPPERCASE_PLACEHOLDER>` in it — those must all be substituted

Begin Phase A pre-flight now. Show every check + result. Wait for user "proceed" before Phase B.
