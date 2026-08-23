# templates/hooks/ — Copilot hook templates, iOS pre-filled (docs/10-MECHANICAL-ENFORCEMENT.md)

| File | Copy to | Event |
|---|---|---|
| `hooks.json.template` | `.github/hooks/framework.json` (commit it) | wiring |
| `hook-io.mjs.template` | `.github/scripts/hook-io.mjs` | shared helpers (payload, state file, quote stripping) |
| `correction-detect.mjs.template` | `.github/scripts/correction-detect.mjs` | `userPromptSubmitted` — Pattern 2 (detect) |
| `doc-freshness-track.mjs.template` | `.github/scripts/doc-freshness-track.mjs` | `postToolUse` — Pattern 5 (track) |
| `lint-fix.mjs.template` | `.github/scripts/lint-fix.mjs` | `postToolUse` — Pattern 4 |
| `stop-gate.mjs.template` | `.github/scripts/stop-gate.mjs` | `agentStop` — Patterns 2 + 3 + 5 (gate) |

Add `.github/hooks/.state/` to `.gitignore`. Fill every `<PLACEHOLDER>` (`<WORKSPACE>`, `<SCHEME>` — or `<PROJECT>` on a
project-only repo — `<PROTECTED_BRANCH>`, `<CHANGELOG_PATH>`, `<STATE_DOC_PATH>`, `<REPO_NAME_FRAGMENT>`;
`<PER_SESSION_PATH>` only if sessions share a checkout). Verify in a FRESH session (docs/10 §
Verification). Pattern 1 (rule surfacing) needs no script — `applyTo:` is native.

**iOS sizing.** `stop-gate.mjs` runs `xcodebuild … build` with a 20-minute internal cap
(`BUILD_CAP_MS`); `hooks.json` gives `agentStop` a 1500-second (25-minute) `timeoutSec`. Keep the
hook timeout ABOVE the cap — timeouts fail open, so the wrong order is a gate that opens precisely
when the build is slow. `lint-fix.mjs` is pre-filled for `swiftformat` on `*.swift` (swap in
`swiftlint --fix` if that is your tool).

If your team also runs the Claude Code edition in the same repo, keep BOTH hook sets: VS Code reads
both `.github/hooks/*.json` and `.claude/settings.json`, but the cloud agent reads only the former
and Claude Code only the latter. The two contracts differ (stdout JSON vs stderr text) — never share
a script between them without an adapter.

`stop-gate.mjs` also takes `<PROTECTED_BRANCH>` and a `RELEASE_HEADING_RE` (the shape of your changelog's release heading, default `## [prod] <date>`): a same-day release heading counts as "docs freshened" even when the entry shipped inside the promote merge (Rule 13).
