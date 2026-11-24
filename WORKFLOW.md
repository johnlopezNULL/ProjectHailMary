# Working method

## 1. Purpose

How a change moves from an ask to the target (the Jetson with the radar and camera) in this repo.
Adapted on November 23, 2026 from the SaaS repo's WORKFLOW.md (adopted there October 17, revised
November 9 and 11, 2026, on the evidence in that repo's `docs/workflow-evidence.md`). The method is
kept; the shop's machinery (staging site, payments, postage, release scripts) is replaced by this
system's equivalents: simulation, bag replay and the hardware bench. AGENTS.md binds; where this
file disagrees with it, AGENTS.md wins. The deep how-to is `docs/engineering-playbook.md`.

## 2. Roles

- **John** decides, approves each gate, merges, runs anything that touches the hardware, and is
  accountable. An agent's approval never counts as his, and no agent message is his consent.
- **The Claude session** plans, writes exact briefs, decides engineering questions, verifies evidence
  and answers John in short messages. It arms no background watches.
- **Opus builders** write code, tests and docs from exact briefs (`.claude/agents/builder.md`).
- **Opus reviewers** review in a fresh context with REVIEW.md (`.claude/agents/reviewer.md`).
- **Sonnet grunts** run commands, apply exact hunks and collect outputs (`.claude/agents/grunt.md`).
- **Codex** (optional) gives an independent adversarial review of High-tier changes.

## 3. The pipeline for every change

**Before the first line:** one message to John with the questions whose answer changes the build
(a threshold, a rate, what counts as a threat, what happens on sensor loss), each with the default
that is built if John does not answer. The build starts on the defaults; nothing waits.

a. **Branch.** A branch (or scratch worktree) off the current `main`. Never commit to, switch, stash
   or reset John's working tree. Never run `git config` in a worktree; use `git -c key=value`.
b. **Tests first for bugs.** A failing GoogleTest (or pytest) case, then the fix. Tests follow the
   intended behaviour and are never weakened; an expectation changes only when it was wrong, with the
   reason written next to it. Each fix is proved by undoing it and watching its test fail; the
   mutations are listed.
c. **Runnable gates, output shown, sized by what changed.**
   - **Code changed** (`src/`, `msgs/`, `config/`, launch files, CMake, the Dockerfile): the gates of
     `docs/engineering-playbook.md` §6: clean build with `-DCUAS_WERROR=ON` (0 warnings), `colcon
     test` + `colcon test-result --verbose` (0 failed, count recorded), cppcheck against its recorded
     baseline, the sim smoke (exits 0, health NOMINAL), `git diff --check`.
   - **Only docs, a harness or a script changed:** prove it (`git diff --quiet <last gated commit>
     <new> -- src msgs` exits 0), run that script's own check, `git diff --check`, and cite the last
     full gates by commit.

   A gate without its output pasted is not passed.
d. **Fresh-context review.** An Opus reviewer against REVIEW.md; fix. At most three rounds; after the
   first, re-review only the new diff. If round three still finds Important issues, stop and
   reconsider the approach.
e. **Pull request**, never a direct push to `main`. Its description has Testing, Risk tier, Contract
   changes (ICD / messages) and How to revert. GitHub Actions (`.github/workflows/build.yml`) builds
   and tests every PR in Docker; the sim smoke and cppcheck are still the author's to run and show.
   A PR that changes behaviour a test pins updates that test, with the reason, in the same PR.
f. **Risk tier** decides what follows (section 4).
g. **Bench run** for High changes (section 11).
h. **Docs.** A `docs/HANDOFF.md` entry at the top; the ICD, architecture diagram and latency budget
   kept true in the same PR as the code that changes them; every number with its conditions and
   date. A docs-only branch is named `docs/<name>` and builds nothing.

## 4. Risk tiers

Set by the change, never by the author. When in doubt, the higher tier. (The playbook's A–D tiers
map onto these: A is a hotfix, section 14; B is High; C is Medium; D is Low.)

- **Low:** docs, comments, formatting, tests only. Merge on the gates of 3c plus the review.
  Exception: a "style" edit that changes control flow, an integer width or a float comparison inside
  the math is High.
- **Medium:** tooling and operator surface: launch files, RViz configs, overlays, health thresholds,
  logging, install scripts. PR plus review plus a sim smoke of the changed view. John merges.
- **High:** anything on the decision path (radar parsing, clustering, clutter map, IMM tracker,
  association and gating, fusion, threat / intent / geofence / reachability, prediction), the
  message contract (ICD, `msgs/`), rates and timing, calibration handling, the CoT output, behaviour
  on sensor loss, sensor configuration, models. Required:
  - the empirical proof of playbook §3 phase 6: every behavioural shape through the real pipeline in
    sim or bag replay, with a readout printing the numbers, plus one pass on production defaults;
  - an independent adversarial review (Codex or a second Opus reviewer) on top of 3d's when the diff
    changes a decision, the contract or timing;
  - John's go for the bench run, and his watched first run on real sensors (section 11);
  - every step that needs John (hardware, a sensor write, a model) named before the work starts.

## 5. Evidence rules

- A claim without output is not done.
- Receipts (counts, hashes, readout lines) for every gate and every bench step, written once under one
  name, each ending with pass or fail.
- An agent's report is verified against its artifacts before it is relayed to John.
- Accept from receipts, never repeat: a step that passed and left a receipt is not run again.
- A claim about ROS, a driver, a library or the hardware needs the installed docs, the header, or a
  probe on the device; never memory.
- Private material (backups, calibration frames, recorded logs) is never committed or sent out.

## 6. Loops and stops

**Bounded self-fix loop** for failures of our own harness (how a script launches, waits for, counts,
reads or records something), never for a finding in the system under test:

- At most 3 retries per step and 12 per run; every script runs under a wall-clock and an idle bound.
- A fix never changes an expectation, lowers a bound, skips a step, reorders steps or removes a
  check. If passing would need that, stop.
- Every fix is logged before the retry and listed in the report.
- The same failure text twice in a row after a fix: stop.
- **A read made too early** (a topic not up yet, a track not yet confirmed) is a harness failure only
  when a later read in the same run shows the expected value with nothing in between that could have
  produced it. A different value (wrong count, wrong class, wrong rate) is always a finding.

**Hard stops, never retried:** any write to a sensor or the board, a model or calibration change, a
merge or push, and every finding in the system under test. A finding is never reclassified as a
harness failure; when in doubt, it is a finding. Never improvise around a failed check: stop, restore,
report.

## 7. Every stop or incident becomes a check

- **A defect, a hardware miss or a process slip** gets a row in the never-again ledger
  (`docs/engineering-playbook.md` §14) and a line in REVIEW.md "Repeated past findings".
- **A stop caused by our own harness or script** gets no ledger row. It becomes an assertion in the
  script or a case in its self-test, so the same cause fails before the next run starts.
- The procedure or script is corrected before the next attempt, never during a run beyond section 6.

## 8. Communication with John

- Lead with the outcome. Short messages.
- One question per gate, and only when his decision changes what happens next.
- When Codex is in the loop, the last thing in the reply is the exact text John pastes into Codex, in
  one code block. A long message goes to a file in the repo; John gets the path, byte size and SHA-256.
- No background watches that wake sessions.
- Decisions are his and are recorded in `docs/HANDOFF.md`.

## 9. Measurement

- A High change's handoff entry ends with one line: review rounds, harness or bench attempts, stops by
  cause (system, harness, procedure), and hours from the last code commit to the bench pass.
- Reverts and incidents are the guardrail. When stops cluster on one cause, fix the method.

## 10. Not adopted, and why

Merge queue, feature flags everywhere, nightly scanning agents: scale machinery for many authors. One
owner and a few changes a week do not need it. (Unlike the SaaS repo, this repo does run CI on every
PR: `build.yml`.)

## 11. Bench runs (this repo's equivalent of the SaaS staging run)

- **The standing bench test, the same for every release:** the sim smoke, the bag replay of the
  recorded shapes, then the live bench with John present: radar and camera attached, the readout open,
  one shape at a time (playbook §7). It changes only in a PR of its own.
- **The feature is proved before the bench** by its unit tests and the sim / replay harness. It gets
  no bench procedure of its own unless its risk can be shown neither offline nor by the standing test
  (a new sensor mode, a new output, timing only real hardware shows); then it is one added step and
  John is told before it is built.
- **John's look is the acceptance.** After the standing test passes, John gets at most five things to
  look at and try, about ten minutes. His answer is quoted in the handoff entry.
- **Nothing on the bench writes to the sensor** unless John says so for that run (AGENTS.md).
- **Budget:** bench setup (harness, procedure) has 4 hours for a High change and 1 hour for a Medium
  one, counted from the last full gates. When tripped, ask John one question: continue with an
  estimate, cut to the decision-path shapes only, or John tests by hand.

## 12. Efficiency rules

- **Facts once.** The first agent on a task writes its facts (hashes, line citations, counts) to a
  file; later agents read it and verify only what they change or doubt.
- **Continue, do not restart.** Fixes and diff-only re-review go to the agent that already holds the
  material.
- **Depth by risk.** Quick: the diff. Standard: the diff and the changed docs and tests. Deep: the
  whole path against the code and the data (anything on the decision path or the contract).
- **Effort by task type.** Builders medium, reviewers high, grunts low; judgment stays on Opus.
- **Fixed output.** Every agent ends with a count and a table; reviewers list only what they would
  block for plus at most five nits.
- **Batch reads**, keep CLAUDE.md lean; the cheapest turn is the one not needed.
- **Checks belong in scripts.** A stop that recurs in prose becomes a script assertion.

## 13. Batches

For work John wants landed together: an integration branch cut from `main`; every part's PR targets
it, each with its own gates and review; the final PR to `main` is John's go and takes the highest tier
of any part.

## 14. Hotfixes

For a defect on the target John wants fixed now (playbook tier A): measure first on the device that
shows it; when John says it was fine before change X, revert X first, then measure. Smallest diff,
gates, rebuild on the target, verify with the readout. One PR per symptom; a second fix for the same
symptom means stop and measure again. Within 24 hours, the bug finder on that diff.

## 15. Status (November 23, 2026)

- Built: this file, AGENTS.md, CLAUDE.md, REVIEW.md, `.claude/agents/`, `docs/review-playbook.md`,
  adapted from the SaaS repo. Not yet used on a change here.
- Already here before: `docs/engineering-playbook.md` (ROS 2 / embedded edition, v1.0), CI.
- Planned by John: replacing ROS 2 with his own runtime. Until it lands, the gates are the ROS ones;
  the pure C++ math classes (no ROS dependency) are the part that carries over, so keep them pure.
