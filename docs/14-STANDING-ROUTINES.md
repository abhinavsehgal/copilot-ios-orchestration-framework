# 14 — Standing Routines (scheduled autonomy)

## The question this chapter answers

Everything before this chapter runs when a person asks. This chapter adds the third leg: **standing
routines** — narrow agent jobs that run on a schedule, produce small reviewable pull requests, and
get *tuned* instead of babysat. For an iOS repo the flagship example is a **crash fuzzer that
drives the real app in the Simulator** — which is why this edition treats routines as a first-class
chapter rather than an appendix.

The pattern is not speculative. The Claude Code team at Anthropic ran it publicly on their own
repos in Aug 2026 — a fleet of daily routines (crash fuzzing on device simulators among them)
opened **388 PRs in a few weeks, 180 merged** after automated review plus human review, at roughly
1-in-50 noise. Nothing in the pattern is vendor-specific; Copilot's cloud agent and headless CLI
are exactly the surfaces it needs.

## Where routines sit

| Leg | Runs when | Chapter |
|---|---|---|
| Interactive orchestration | a person asks | 02, 03, 06 |
| Mechanical enforcement | a session does something | 10 |
| **Standing routines** | **the clock says so** | **this one** |

A routine is *not* a cron script with an LLM inside. It is a specialist (Chapter 3) with a written
charter, an output contract, a review gate, and a tuning log. If a job needs no judgment (bump a
build number, rotate provisioning), use plain Actions/Fastlane automation; a routine earns its cost
only where the work needs reading, reasoning, or writing code.

## The seven conventions

1. **One routine, one narrow charter** — checked in as a file
   (`templates/routine.md.template`), diffed and reviewed like code. A charter that needs "and" is
   two routines.
2. **Output is small, self-contained PRs — never direct pushes.** The cloud agent's PR-per-task
   shape already enforces this.
3. **Every fix-PR carries a repro and a truth table** — for a crash fix, the exact tap/gesture
   sequence or the crash log that reproduces it; for a behavior change, before/after at the
   affected call sites, verified in the Simulator on the real app, not a mock. "No evidence, no
   claim" tightens when nobody is watching.
4. **All routine activity reports to one place** — one issue thread or channel per fleet. A run
   that produces nothing still posts; silence must mean broken, not idle (Pitfall 42).
5. **A routine never merges its own PR.** Copilot code review on every routine PR; a stronger
   review pass for anything touching payments, auth, entitlements, or privacy surfaces; a human on
   every merge.
6. **Wrong output tunes the routine, not just the output.** Patch the charter, date the change in
   its tuning log. Expect ~1-in-50 noise when tuned; retire a routine still over its noise budget
   after two tuning passes.
7. **Attempt caps and a verified retire path.** Per-run budget, park-after-N-failures, and a
   *checked* completion write — an unattended loop whose "done" write fails unread will grind
   forever (Pitfall 42; one production pipeline did exactly this for 17 days).

## The iOS starter catalog

- **Simulator crash fuzzer** (the flagship) — boot the app in the Simulator (`xcrun simctl`),
  drive it through real flows plus randomized taps/gestures (XCUITest is the native harness),
  collect crash logs, root-cause, open a fix PR with the repro sequence and the symbolicated trace
  attached. No mocks: the routine exercises the same binary a tester would.
- **Privacy-drift-weekly** — the REVIEW-ONLY `ios-privacy` agent (Chapter 3) on a clock: diff
  `Info.plist` usage strings, entitlements, and the privacy manifest against the documented
  nutrition labels; open an *issue* (REVIEW-ONLY agents never write) when they diverge. Catching
  this weekly beats catching it in App Review.
- **Dead-asset sweeper** — unreferenced images/colors in asset catalogs, orphaned localization
  keys, `#available` branches whose floor the deployment target has passed. For *suspected*
  dead code paths: add a log first, confirm silence next run, remove then.
- **Flaky-UI-test root-causer** — pick the week's flakiest XCUITest, find the root cause (timing,
  animation waits, shared state), fix or delete with justification. Never auto-retry as the fix.
- **Duplicate-abstraction unifier** — similar-yet-divergent views/helpers unified behind one, with
  a truth table over call sites.
- **Doc-drift checker** — Tier-1/Tier-2 docs vs reality (dead links, stale build commands, a
  `PROJECT.md` §3 whose TestFlight build column went stale).
- **Contract-drift checker** (workspaces) — see Chapter 13 § Scheduled workspace routines: the
  app is a consumer of every contract, so drift lands here first; include shared Swift packages
  (pinned by tag) in the diff.
- **Hill-climber** — one metric per routine via
  `templates/skills/hill-climb/SKILL.md.template`: cold-launch time, app size, build wall-clock,
  test-suite duration.

## Running them

1. **Scheduled workflow → cloud agent.** A GitHub Actions `schedule:` workflow creates/re-opens
   the routine's tracking issue and assigns it to Copilot. Simplest where the routine's work is
   code-reading and PR-writing.
2. **Scheduled workflow → headless CLI on a macOS runner.** Anything that must *run the app*
   (the crash fuzzer, UI-test triage) needs the Simulator, and the Simulator needs macOS —
   schedule `copilot -p --agent=<routine-agent>` on a `macos-*` runner with the Xcode version
   pinned. Budget runner minutes per routine; macOS minutes are the expensive kind.
   Note: whether `.github/hooks/*.json` fire in headless CI sessions is **not re-verified** — test
   in your install and stamp a verified-on date before relying on hook enforcement inside routine
   runs; until then the PR gate (convention 5) is the enforcement layer.
3. **Org-level automation** — same charter and gates; the mechanism changes, the conventions do
   not.

Every mechanism gets a kill switch a human can flip without a deploy (disable the workflow, close
the tracking issue with a label).

## What NOT to do

- Don't let a routine merge, push to the integration branch, or touch signing/release lanes — PRs
  only, gates always.
- Don't launch a fleet before the first routine has run for two weeks and had its noise tuned down.
- Don't give a routine agent broad `tools:` "to be safe" — charter-narrow, like any specialist;
  and keep privacy/compliance routines REVIEW-ONLY (issues, not PRs).
- Don't run a routine without a budget and an attempt cap — a runaway loop on macOS runner minutes
  is the expensive version of the canonical failure (REFINEMENT check A12).
- Don't report "ran" as "worked" — completion means the verified completion write landed.
- Don't fuzz against mocks or a stub backend and call the result a crash pass — the charter says
  which environment counts, and the repro must reproduce there.

## Cross-links

- `templates/routine.md.template` — the charter file every routine checks in.
- `templates/skills/hill-climb/SKILL.md.template` — the metric-loop skill routines can wrap.
- `docs/06-INVOCATION-MODES.md` § Scheduled runs — where routines sit among the surfaces.
- `docs/10-MECHANICAL-ENFORCEMENT.md` — the correction-capture pattern is the tuning loop's
  interactive twin.
- `docs/12-PROJECT-TRUTH-AND-LEARNINGS.md` — routine tuning logs follow the LEARNINGS entry shape.
- `docs/13-MULTI-REPO-WORKSPACES.md` § Scheduled workspace routines — the contract guardian on a
  clock.
- `docs/08-IOS-COMMON-PITFALLS.md` — Pitfalls 41 (context weight) and 42 (unattended jobs need a
  verified retire path).
