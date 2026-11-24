# Testing handbook

Written November 23, 2026, for John, who owns testing and releases on this project. It explains how
testing works here and what each practice is called at work. It teaches; it does not bind.
AGENTS.md and WORKFLOW.md bind, and where this file and they disagree, they win.

Each section says what the practice is in plain words, then **On this project today** (facts checked
against the repo on November 23, 2026), **Planned** where something does not exist yet, and one
**At work** line on how a team usually does it. Terms in bold are defined in §12. The lessons behind
many rules are in `docs/lessons-ledger.md` (rows CL-01 to CL-48).

## 1. The path a change takes

A change passes a series of checks. The early ones are cheap and fast and catch most mistakes. The
late ones are slow and close to real use and catch what the early ones cannot. Each stage should
only have to find what the stages before it could not.

| # | Stage | The question it answers | On this project today |
|---|---|---|---|
| 1 | **Unit test** | Does one class or function do the right thing on its own? | 22 GoogleTest programs: 16 in `test/unit/`, 6 in `src/cuas_fusion/test/`, all listed in `src/cuas_fusion/CMakeLists.txt` |
| 2 | **Integration test** | Do several real parts work together? | `src/cuas_fusion/launch/sim.launch.py` (a simulated radar into the real nodes) and `replay.launch.py` (a recorded **bag**, a ROS 2 recording of sensor messages, into the real nodes); no checked-in script judges their output yet |
| 3 | **Gates** | Did anything basic break? | WORKFLOW.md §3c: build, unit tests, cppcheck, the **sim smoke** (the full pipeline run for a short time on the simulated radar), `git diff --check` (§8 here) |
| 4 | **Code review** | Is the change right, and what does it break that no test covers? | A fresh reviewer agent with REVIEW.md (`.claude/agents/reviewer.md`) |
| 5 | **Pull request** and **CI** | Does it build and pass on a clean machine, and does John accept it? | `.github/workflows/build.yml` on every PR into `main`; John merges |
| 6 | **Bench run** | Does it work on the real Jetson, radar and camera? | WORKFLOW.md §11, John present (§9 here). This is this project's **system test** on real hardware and its **staging**; John's look is the **acceptance test** |
| 7 | Release | Is the merged build right on the target? | Rollback written before merge, first run on real sensors watched (playbook §8 rules 7, 8) |
| 8 | After a problem | How do we stop it from happening again? | A **regression test** plus a ledger row (WORKFLOW.md §7; playbook §14) |

**Why the order matters.** A unit test finds a bug in seconds on any machine. The bench costs John's
time and real hardware. If a bug can be caught at stage 1, catching it at stage 6 wastes the most
expensive time on the project.

**At work:** teams run the same ladder as a CI pipeline: unit and integration tests on every push,
slower suites before merge, then a staging environment before a release.

## 2. Keeping testing fast

The **test pyramid**: many unit tests that run in seconds, fewer integration and replay tests that
run in minutes, and very few bench runs that cost the owner's time. Speed matters because a test
that answers in seconds gets run after every edit, so a mistake is caught while it is still fresh.
That is a **fast feedback loop**. A test that takes an hour gets skipped or batched, and bugs pile
up behind it. Moving checks to the earliest, cheapest stage is called **shift-left**.

**On this project today:**

- **Tests are sized by what changed** (WORKFLOW.md §3c). A change to code runs the full gates. A
  change to docs, a harness or a script first proves that no product code changed, then runs only
  its own check. This is **change-based test selection**.
- **A feature is proved offline and gets no rig of its own** (WORKFLOW.md §11, §16). Its unit tests
  and the sim or replay run carry the proof. The bench runs the same standing test for every
  release.
- **Test-tool work is time-boxed** (**time-boxing**). Bench setup has 4 hours for a High change and
  1 hour for a Medium one (WORKFLOW.md §11). Test-tool work for a change is also capped at the time
  spent on the change's own product code and tests; a change whose product is a test tool gets its
  budget in its brief (WORKFLOW.md §17). Whichever limit trips first, John gets one question with
  three answers: continue with an estimate, cut to the decision-path shapes only, or John tests by
  hand.
- **It is measured on every change** (WORKFLOW.md §9): the wall-clock time of each stage it ran
  (unit, replay, gates, bench) and, when there was test-tool work, the hours on test tools beside
  the hours on the product.
- **Speed targets** (WORKFLOW.md §17) were set on November 23, 2026 and are not measured yet: unit
  tests under 1 minute, replay under 10 minutes, full gates under 20 minutes, a bench run at most 30
  minutes of John's time. The only times measured so far are builds on the Jetson during the
  September 2, 2026 audit: two builds of both packages took 4 min 57 s and 5 min 41 s, and an
  incremental build took 50 s (`docs/audit-log.md`). The test run, the full gates and CI have not
  been timed.

**Why this is a rule here.** On the earlier web-shop project, one small feature took about 3 hours
of product work and about 13 hours of test machinery written for it alone
(`docs/lessons-ledger.md` CL-33). Section 17 of WORKFLOW.md exists so that cannot happen here.

**At work:** teams watch how long CI takes and how often it fails for no product reason, and split
fast checks (every push) from slow ones (nightly or before release).

## 3. From a bug to a regression test

The steps, in order:

1. **Reproduce.** Make the bug happen on demand: a unit-test input, a sim scene or a bag and a
   timestamp. On a device-only symptom, measure on that device first (WORKFLOW.md §14).
2. **Write the failing test.** It states the intended behaviour, not what the code does today.
3. **Fix.** The smallest change that makes the test pass.
4. **Undo the fix and watch the test fail.** This proves the test can catch the bug: it is the
   "red" step of **test-driven development**, done after the fact, or one hand-made mutant (a
   **fail-without-the-fix check**). A test that still passes with the fix undone tests nothing
   (WORKFLOW.md §3b).
5. **Keep the test.** It is now a regression test that guards the fix forever.

**Worked example from this repo.** The September 2026 audit found that `FusedDetection.range_m`
held the forward coordinate instead of the real distance (`docs/audit-plan.md`, row RC-29).

- The test is `RangeIsEuclideanNotForwardCoordinate` in `test/unit/test_fusion_engine.cpp` (line 74).
  A target at (-2, 10, 2) m is √(4 + 100 + 4) = √108 ≈ 10.39 m away, not 10 m.
- The expected value, √108, comes from hand arithmetic, not from the code under test. That is what
  makes it a real check (see the **test oracle**, §4).
- The fix is one line in `src/cuas_fusion/src/fusion/fusion_engine.cpp` (line 169): the range is
  √(x² + y² + z²).
- The fail-without-the-fix check: set `range_m` back to the forward coordinate, run
  `colcon test --packages-select cuas_fusion --ctest-args -R test_fusion_engine` (**colcon** is the
  ROS 2 build and test tool), see this test
  fail, then restore the line. The audit record does not say whether this step was run at the
  time; this is how to run it now.

**At work:** every bug ticket is closed with a link to the test that reproduces it.

## 4. Designing a test to find problems

A good test is written to break the system, not to show it works.

- **Start from what the system must do**, not from the code: the message contract (the **ICD**, interface control document, `docs/ICD.md`),
  the claims checked in `docs/claims-evidence.md`, and John's decisions (`docs/HANDOFF.md`).
- **Equivalence partitioning:** split the inputs into groups that should behave alike, and test one
  of each. For tracks: none, one, a few, a full table.
- **Boundary value analysis:** bugs cluster at edges, so test 0, 1, max and max+1. Here the track
  table holds `TRACK_MAX_TRACKS` = 32 tracks and a radar frame at most `RADAR_MAX_POINTS_PER_FRAME`
  = 256 points (`src/cuas_fusion/include/cuas_fusion/common/constants.hpp`, lines 52 and 57).
- **The test oracle** is where the expected answer comes from. It must not be the same computation the
  test checks: use hand arithmetic, a reference implementation, a recorded output of a trusted run
  (a **golden file**) or the ICD. A test that copies
  the code's own formula passes even when the formula is wrong.

**Edge cases for this system** (REVIEW.md "Lenses" lists several):

| Case | What must happen | On this project today |
|---|---|---|
| No targets | Empty frames are still published, and health does not call the sensor dead | Fixed in the audit (`docs/audit-plan.md` RC-12); `IdleKeepsOkAndDeadReportsZeroHz` in `src/cuas_fusion/test/test_health_monitor.cpp` |
| One detection | A single clutter return never becomes a confirmed track | REVIEW.md "What failure looks like in this repo" (track identity) |
| Two targets crossing | Track ids do not swap | The `crossing` scene in `sim.launch.py` |
| Sensor goes silent | Last-known-good plus one log line; health shows the loss | Playbook §1 principle 6; the health monitor |
| Malformed frame | A cut, oversized or impossible frame is rejected | Playbook §4 role 5, row S1 |
| 32 tracks | The table fills without overflow; target 33 is handled, not lost silently | `TRACK_MAX_TRACKS` above |

**At work:** test cases are written from the requirements and reviewed like code.

## 5. Fakes and simulation

A **test double** stands in for a real part during a test. The standard vocabulary (from the book
*xUnit Test Patterns*) names five kinds. A **dummy** is passed in but never used. A **stub** returns
fixed answers. A **fake** is a simple working version. A **spy** records how it was called, for the
test to check afterwards. A **mock** is told in advance which calls to expect and fails the test
itself when they do not come.

**On this project today:** `sim_radar_node` is a fake radar (`src/cuas_fusion/src/drivers/`). It
plays one scripted scene (`approach`, `lateral`, `circle`, `two_targets`, `crossing`, `clutter`,
`max_range`, `hover`) at 20 Hz with a noise seed of 42 by default (`sim.launch.py`). No mocking
library is used.

**A fake must match the real sensor's timing and failure behaviour.** A fake that is faster, cleaner
or differently oriented than the real sensor hides exactly the bugs the real sensor will show. This
happened here: the sim scenes treated +X as forward, but the ICD's `radar_frame` has Y as range, so
fusion published nothing in every sim smoke and nobody asked why (playbook §14;
`docs/audit-log.md`). That is the **sim-to-real gap**, and closing it is **test double fidelity**.

**The fake-versus-real table** (WORKFLOW.md §16) sets each behaviour a test judges beside the real
sensor's. A start for the radar, from the code; the blanks are filled from a recorded bag:

| Behaviour | Sim today (`src/cuas_fusion/src/drivers/sim_radar.cpp`, `sim_radar_node.cpp`) | Real radar |
|---|---|---|
| Frame period | 50 ms wall timer (20 Hz) | 50 ms in the profile's `frameCfg` (REVIEW.md) |
| Frame jitter, dropped frames | Not modelled | To measure from a bag |
| Startup delay | Not modelled | To measure on the bench |
| Points per target | One centroid per target | Clusters after the parser; to measure |
| Position noise | Seeded Gaussian, σ = 0.10 m (`kSimRadarNoiseSigmaM`) | To measure |
| Frame convention, Doppler sign | X = azimuth, Y = range; sign fixed in the audit | X = azimuth, Y = range (`docs/HANDOFF.md`) |

**Software in the loop (SIL) and hardware in the loop (HIL).** SIL runs the real software on a
desktop against a simulation of its sensors: the sim on the desktop. HIL puts the real target
hardware in a loop with a real-time simulator that feeds its sensor inputs (a radar emulator on the
serial port, for example); this project has no HIL rig. The scenes here are scripted and do not
react to the system's outputs, so they are **open loop**, and replaying a bag is **replay testing**.
The bench, with real sensors watching real targets, is a **system test** on the target hardware.

**At work:** teams keep a HIL rig for hardware timing and a SIL suite that runs on every change.

## 6. Timing and flaky tests

A **flaky test** sometimes passes and sometimes fails on the same code. The most common cause is a
fixed sleep: the test waits a guessed time and then reads. On a slow day (a cold TensorRT engine, a
busy Jetson) the guess is too short and the test fails although the product is fine; on a fast day
it wastes time. The cure is an **explicit wait**: poll, with a time limit, for the system's own
signal (the message arriving, the track reaching CONFIRMED, the node reporting ready).

**On this project today:**

- `scripts/play_and_echo.sh` starts the bag, sleeps 4 s (line 13), then subscribes. That is a guessed
  wait, and it also creates a **late-joiner problem**: every topic here is Volatile (`docs/ICD.md`),
  so anything published in those 4 s is lost. Its next part is right: it polls for up to 20 s for the
  first message, which is a bounded wait. `scripts/check_outputs.sh` also polls, for up to 15 s.
- Not every sleep is wrong. `ConcurrentAddAndFindNoDataRace` in
  `test/unit/test_timestamp_associator.cpp` sleeps 1 ms per loop to interleave two threads. That is
  pacing, not a wait before a read. Under REVIEW.md's rule it should carry its reason beside it.
- **The slow leg** (WORKFLOW.md §16): a harness's self-test includes a run in which every dependency
  answers after at least 1.5 s, so a race shows up offline instead of on the bench.
- **When may a failed read be called "too early"?** Only when all four conditions of WORKFLOW.md §6
  hold, and never for a threat level, geofence state, track identity or anything on real hardware.
  That retry is switched off until the harness records an **action timeline**. Until then, a failed
  read is always a finding.

**At work:** teams track flaky tests, take them out of the required set while they are fixed
(quarantine), and never "fix" a flaky test by retrying it until it passes.

## 7. Test tools are code too

Test tools are code: a harness, a replay script, a **readout** (a node or script that prints the numbers a test judges), a test driver (WORKFLOW.md §16 has the full definition). A wrong one stops good runs or
hides real bugs. So a test tool gets its own test: a **self-test** that runs it against fakes with
known answers, including the slow leg.

**On this project today:**

- No harness or self-test is checked in yet. The audit's harness scripts were gitignored helpers
  (named in `docs/HANDOFF.md`). The scripts in `scripts/` are not checks. `check_outputs.sh` and
  `play_and_echo.sh` print what they see and judge nothing; `send_radar_config.sh`, `test_radar.py`
  and `run_radar.sh` write to the radar, and `calibrate_camera.py` writes a calibration file, so
  none of those runs in a test (AGENTS.md).
- **Extend the tool that last passed** (WORKFLOW.md §16). A new feature adds a mode or a scene; it
  never gets a new rig. This is **shared test infrastructure**.
- **One review round** on a test tool's first complete version. Every later fix is proved by a
  self-test case that fails without the fix, not by another review (WORKFLOW.md §3d). This is
  **testing the test code**.

**Planned:** one shared test bench: one sim source, one replay harness, one readout and one bench
procedure, each with a self-test (WORKFLOW.md §15 says it is not built yet).

**At work:** test infrastructure is owned like product code: reviewed, versioned and tested.

## 8. Gates and CI

A **gate** is a check that must pass before the next step. **CI** (continuous integration) is
the practice of merging every change into `main` often, each change checked by an automated build
and test run.

**On this project today, the gates for a code change** (WORKFLOW.md §3c):

| Gate | Pass today |
|---|---|
| `colcon build` with `-DCUAS_WERROR=ON` | 0 warnings (warnings fail the build) |
| `colcon test` and `colcon test-result --verbose` | 0 failed, count recorded (160 GoogleTest cases at the September 2026 audit, `docs/HANDOFF.md`) |
| cppcheck | Against its recorded baseline of 6 findings (`README.md`) |
| The sim smoke | No checked-in script yet, so each run's pass line names what it read: topics and rates (`ros2 topic hz`, over how long), the `status` field of `/health/status`, and the allowed log lines by exact text. `docs/HARDWARE_TEST_PLAN.md` §1 is an older manual check for one branch and judges other things (nodes start, no FATAL lines, `/tracks` publishes) |
| `git diff --check` | No whitespace errors |

`docs/engineering-playbook.md` §6 still shows placeholders for the coding-standard checker and the
sim smoke: their literal commands are not written down yet.

**CI** (`.github/workflows/build.yml`) runs on every pull request into `main` and every push to
`main`. It builds the Docker image (`ros:humble`, with `-DCUAS_WERROR=ON -DCUAS_BUILD_GPU=OFF`, so the
TensorRT detector and `inference_node` are not built), then runs `colcon test --packages-select
cuas_fusion` and `colcon test-result --verbose`. It does not run cppcheck or the sim smoke; those
stay the author's to run and paste (WORKFLOW.md §3e). Its run time is not measured yet.

**What "pass" must name** (WORKFLOW.md §3c). A pass line says what was compared, how, and which
failures are allowed, by exact text. "Exit 0" or "NOMINAL" alone is not a pass.
`scripts/check_outputs.sh` shows why: it prints "WARNING: No threat reports received" and still
exits 0. A pass line in the right shape: `colcon test-result --verbose on <commit>: <N> tests,
0 errors, 0 failures, 0 skipped; allowed failures: none`. These are **pass/fail criteria**.

**At work:** CI is a required check on every PR, and `main` is kept green.

## 9. The bench run

The bench is this project's staging and its system test on real hardware: the Jetson with the real radar and camera, John present. It is
the most expensive test, so it runs one standing procedure, the same for every release.

**On this project today** (WORKFLOW.md §11; playbook §7):

- The order: the sim smoke, then the bag replay of the recorded shapes, then the real sensors one
  shape at a time with the readout open.
- **John's look is the acceptance test**: at most five things to look at and try, about ten
  minutes; his answer is quoted in the handoff entry.
- **Nothing writes to a sensor without John's word** (AGENTS.md). Note three traps:
  `src/cuas_fusion/launch/full.launch.py` runs `scripts/send_radar_config.sh` every time it starts,
  `scripts/test_radar.py` sends the full radar config, and `scripts/run_radar.sh` runs
  `test_radar.py` before it starts the parser. All three are sensor writes. Read-only is fine:
  listing devices, `ros2 topic hz`, reading the parameter dump.
- The Cursor-on-Target output stays on this machine during a test (AGENTS.md).
- **First contact with real hardware is a read-only probe** (WORKFLOW.md §16).

**Planned:** a **bench manifest**: a checked-in list of what the bench should be (firmware and
software versions, model, engine and calibration hashes, which sensor is on which port), checked
read-only at the start of every run; drift stops the run (`docs/lessons-ledger.md` CL-10).

**At work:** a staging or hardware rig mirrors production, and a named person signs off before
release.

## 10. Evidence

- **A claim without output is not done** (WORKFLOW.md §5). Every gate and bench step leaves a
  **receipt**: the command, its output, and pass or fail.
- **Every number carries its conditions and date**: sim or live, which sensor, which board, model
  precision, clock source (AGENTS.md; playbook §8 rule 6). An unmeasured number is worded as
  unmeasured.
- **Measure on the device that shows the problem** (WORKFLOW.md §14). The desktop build agreeing
  proves nothing about a symptom only the Jetson shows.
- **Compare the candidate with the base build, measured the same way**: same bag, launch and clock
  (**differential testing**; `docs/lessons-ledger.md` CL-05).

**On this project today:** `docs/claims-evidence.md` is the facts sheet (each claim, its evidence,
its confidence), and `docs/audit-log.md` is an append-only record with UTC times. An example of why
this matters: the radar was documented at 16 Hz, but its profile's 50 ms frame period gives 20 Hz
(REVIEW.md "Repeated past findings").

**At work:** test reports and CI logs are attached to the PR, so a reviewer reads results, not
promises.

## 11. Not here yet

Practices a junior engineer is expected to know, and how far this project uses each one today.

| Practice | What it is | Does it fit here, and when? |
|---|---|---|
| Test levels by name | Unit, integration, system, acceptance | Now: §1 of this file names them |
| **Code coverage** | Share of lines or branches the tests run | Yes: gcov or lcov on the pure C++ libraries in CI, number recorded, no threshold at first (`docs/STANDARDS_CHECKLIST.md` D7) |
| **Sanitizer** | Compiler checks for memory and threading bugs while tests run | Yes, soon: a CI job on the unit tests; `test_timestamp_associator.cpp` already asks for a ThreadSanitizer run by hand |
| **Fuzzing** | Random or mutated input, to find crashes | Yes: the radar frame parser reads untrusted bytes; with the ROS-free runtime's parser |
| **Property-based testing** | A rule checked on many generated inputs | Yes: tracker invariants (the covariance stays valid; `test/unit/test_imm_filter.cpp` checks it over 2000 steps) |
| **Mutation testing** | A tool changes the code in many small ways and reports how many changes the tests catch | Later, on the pure math libraries; today only the hand-made fail-without-the-fix check (§3) |
| **Static analysis**, deeper | Tools that read code for bugs | cppcheck is a gate; clang-tidy was run for `docs/AUDIT_REPORT.md` but is not a gate and has no config file in the repo |
| **Performance regression test** | Fails when a candidate is slower than the base | Yes: a latency percentile in the planned readout, against `docs/latency_budget.md` |
| **Requirements traceability** | Requirement to code to test | Lightly: `docs/STANDARDS_CHECKLIST.md` D6; `docs/claims-evidence.md` is a small version |
| **Test plan** and **test report** | What will be tested; what happened | `docs/HARDWARE_TEST_PLAN.md` is an older plan for one branch; bench receipts become the report |
| **Bug tracker** | Open bugs with owner and status | Lightly: GitHub Issues with labels; today decisions live in `docs/HANDOFF.md` |
| **Golden file** tests | Saved outputs compared with tolerances | Yes: one per recorded bag, with the replay set |
| Mocking library (gMock) | Mocks without hand-written fakes | When a test needs to check calls; not needed yet |
| Real-time terms (WCET, deadline, jitter) | Worst-case time a step may take | With John's runtime scheduler (`docs/STANDARDS_CHECKLIST.md` D1) |
| Pinned dependencies | Exact versions of images and libraries | Yes: the Dockerfile uses the `ros:humble` tag, not a pinned digest |
| Formatting tools and pre-commit hooks | clang-format before each commit | Yes, cheap; not set up |

## 12. Glossary

Every industry term used in this file and in `docs/lessons-ledger.md`.

| Term | Plain meaning | Where it shows up here |
|---|---|---|
| acceptance criteria | The conditions a feature must meet for its owner to accept it, agreed before it is built | John's five things to look at and try (WORKFLOW.md §11) |
| acceptance test | A test of whether the system does what the owner asked, judged by the owner | John's look on the bench (WORKFLOW.md §11) |
| action timeline | A record of what a harness did and when, beside what it read and when | Planned; WORKFLOW.md §6 keeps its retry off until it exists |
| adversarial review | A review that tries to break the change rather than approve it | REVIEW.md; `docs/review-playbook.md` |
| allow-list | A fixed list of what is permitted; anything else is a finding | A shared list of harmless warnings (Planned; CL-12, CL-42) |
| bag (rosbag) | A ROS 2 recording of messages on chosen topics, played back later | `replay.launch.py`; `scripts/play_and_echo.sh` |
| baseline capture | Recording how the base build behaves, to compare a candidate against it | cppcheck's baseline; CL-42 |
| bench manifest | A checked-in list of what the bench should be, checked at the start of every run | Planned (CL-10) |
| bench run | This project's staging run on the real hardware, John present | WORKFLOW.md §11 |
| bisection | Finding the breaking change by testing halfway points in history (`git bisect`) | CL-46 |
| boundary value analysis | Testing at the edges of a range: 0, 1, max, max+1 | §4; `TRACK_MAX_TRACKS` |
| brittle assertion | A check that breaks for reasons unrelated to what it tests, such as matching free text | CL-23 |
| bug tracker | A list of open bugs with owner and status | Not here yet (§11) |
| build provenance | Knowing exactly which commit and settings produced the running program | CL-25; playbook §9.3 |
| cache invalidation | Throwing a stored copy away when what produced it changes | TensorRT engines (CL-20) |
| cache staleness | A stored copy older than the truth | CL-22 |
| change-based test selection | Running only the checks a change can affect | WORKFLOW.md §3c |
| clean build | A build from an empty build folder in a clean checkout | Playbook §6 pre-step; CI; CL-47 |
| code coverage | The share of code the tests run | Not here yet (§11) |
| code review | Another person or agent reading the change before merge | REVIEW.md |
| colcon | The ROS 2 command-line tool that builds and tests the packages | `colcon build`, `colcon test` (§8) |
| configuration drift | The real setup slowly differing from the record | CL-10, CL-39 |
| continuous integration (CI) | The practice of merging every change into `main` often, each change checked by an automated build and test run | `.github/workflows/build.yml` runs the build and tests; CL-18 |
| definition of done | The written list of what must be true before work counts as finished | Playbook §13 |
| deterministic test | A test that gives the same result every run on the same input | `test/unit/test_imm_filter.cpp` (a fixed `std::mt19937` seed); the sim's seed fixes its noise, not its wall-clock timing |
| differential testing | Running two versions on the same input and comparing the outputs | §10; CL-05 |
| documentation drift | Docs that no longer match the code or the measurements | The 16 Hz radar rate (CL-24) |
| draft pull request | A PR marked as not ready to merge | CL-45 |
| dry run | Running a procedure in a mode that writes nothing, to find mistakes first | CL-11 |
| dummy | A stand-in that is passed in but never used | §5 |
| environment parity | How closely the test machine matches the real one | Desktop build versus the Jetson (CL-09) |
| equivalence partitioning | Splitting inputs into groups that behave alike and testing one of each | §4 |
| explicit wait | Waiting for a named condition with a time limit, instead of a fixed time | §6; WORKFLOW.md §16 |
| failure artifacts | Everything saved at the moment a test fails: logs, last messages, settings | Planned (CL-41) |
| fail-without-the-fix check | Undoing a fix to prove its test fails without it: one hand-made mutant | §3; WORKFLOW.md §3b; CL-07 |
| fake | A simple working stand-in for a real part | `sim_radar_node` |
| fake-versus-real table | Each behaviour a fake must copy, beside the real thing's | §5; WORKFLOW.md §16 |
| fast feedback loop | Getting a test result soon enough to act on it at once | §2; WORKFLOW.md §17 |
| flaky test | A test that sometimes passes and sometimes fails on the same code | §6 |
| flaky-test triage | Deciding by fixed rules whether a failure is the test's or the product's | WORKFLOW.md §6 |
| fuzzing | Feeding many random or mutated inputs to find crashes | Not here yet (§11) |
| gate | A check that must pass before the next step | WORKFLOW.md §3c |
| golden file | A saved output that later runs are compared against | Not here yet (§11) |
| graceful degradation | When an input or part fails, falling back to a safe, reduced mode (last-known-good) instead of crashing or passing on bad data | Playbook §1 principle 6; CL-29 |
| hardware in the loop (HIL) | The real target hardware runs the real software while a real-time simulator feeds its inputs and reacts to its outputs | Not here yet; the bench is a system test (§5) |
| harness | The code that drives the system under test and reads its results | The sim and replay launches; the planned bench |
| hermetic test | A test that depends on nothing outside itself: no real network, no real device | CL-15 |
| hypothesis-driven debugging | Treating each suspected cause or fix as a guess to measure | Playbook §5 |
| ICD | Interface control document: the contract that defines every topic, message, unit and rate | `docs/ICD.md` |
| idempotency | Doing something twice has the same effect as doing it once | CL-30 |
| integration test (integration testing) | A test of several real parts working together | §1 |
| keep the build green | Never leaving `main` or an integration branch with failing tests | WORKFLOW.md §13 |
| late-joiner problem | A listener that starts after messages were sent misses them | §6; CL-03 |
| lean documentation | Writing only the records someone will use | WORKFLOW.md §3h |
| line-ending normalization | Making files use one line ending (LF or CRLF) so hashes and scripts agree | CL-19 |
| mock | A stand-in told in advance which calls to expect; it fails the test itself when they do not come | Not used here |
| mutation testing | A tool makes many small changes (mutants) to the code, such as `<` to `<=`, and reports how many the tests catch (the mutation score) | Not here yet; §3's check is one hand-made mutant |
| observability | How well the system's own logs, counters and stamps show what it did | CL-41; the health monitor |
| off-by-one | An error at the exact edge of a range or a count | CL-27 |
| open loop | A test whose inputs are scripted in advance and do not react to the system's outputs | The sim scenes and bag replay (§5) |
| over-specified test | A test that demands more than matters, such as exact equality on things it does not control | CL-17 |
| pass/fail criteria | The written rule, set before a test runs, that decides whether it passed | WORKFLOW.md §3c pass rule; CL-08 |
| performance regression test | A test that fails when the candidate is slower than the base | Not here yet (§11) |
| poka-yoke | Japanese for mistake-proofing: making a mistake impossible or caught at once | WORKFLOW.md §7 |
| production parity | Testing with the same settings and defaults the real system runs | WORKFLOW.md §4 production-defaults pass |
| property-based testing | Checking a rule on many generated inputs | Not here yet (§11) |
| quarantine | Taking a flaky test out of the required set while it is fixed | §6 |
| pull request (PR) | A proposed change on a branch, reviewed before merge | WORKFLOW.md §3e |
| readout | A node or script that prints the numbers a test judges, in a fixed format | Planned; `scripts/check_outputs.sh` is the nearest today |
| receipt | A saved record of one check's output, ending in pass or fail | WORKFLOW.md §5 |
| regression test | A test that keeps a fixed bug from coming back | `RangeIsEuclideanNotForwardCoordinate` (§3) |
| removed-behaviour audit | Listing every guarantee old code gave before moving or deleting it | REVIEW.md "Lenses" |
| replay testing | Feeding recorded sensor data to the real software and checking its outputs | `replay.launch.py` |
| reproduce before fixing | Making the bug happen on demand before changing code | §3; WORKFLOW.md §14 |
| requirements traceability | Linking each requirement to its code and its test | Not here yet (§11) |
| revert first | Undoing the suspected change before trying new fixes | WORKFLOW.md §14 |
| runbook validation | Running every step of a written procedure as it will really run, before handing it over | CL-26 |
| safety interlock | Code that refuses a dangerous action unless the real conditions hold | CL-16 |
| sanitizer | A compiler option that checks memory or threading while the program runs | Not here yet (§11) |
| self-test | A test tool's own test, run against fakes with known answers | §7; WORKFLOW.md §16 |
| shared test infrastructure | One set of test tools that every feature reuses | WORKFLOW.md §16 |
| shift-left | Moving checks to earlier, cheaper stages | §2; WORKFLOW.md §11 |
| sim smoke | This project's smoke test: the full pipeline run for a short time on the simulated radar | WORKFLOW.md §3c; §8 |
| sim-to-real gap | The differences between a simulation and the real world | §5; CL-02 |
| single source of truth | Each fact or decision has exactly one home | Playbook §1 principle 5; CL-21 |
| slow leg | A self-test run where every dependency answers after at least 1.5 s | §6; WORKFLOW.md §16 |
| smoke test | A quick run that checks the system starts and does the basics | The sim smoke (§8) |
| soak test | A long run that looks for slow problems such as memory growth | Playbook §8 rule 14 |
| software in the loop (SIL) | The real software, on a desktop, run against a simulation of its environment instead of hardware | `sim.launch.py` on the desktop (open loop) |
| specification by example | Agreeing on behaviour through concrete examples | Playbook §2; CL-36 |
| spy | A stand-in that records how it was called, for the test to check afterwards | Not used here |
| staging | The last test place before real use, as close to real as possible | The bench (§9) |
| static analysis | Tools that find bugs by reading code without running it | cppcheck (§8) |
| stub | A stand-in that returns fixed answers | Not used here |
| system test | A test of the whole, integrated system against its requirements, in an environment as close to real as possible | The bench run (§9) |
| test double | Any stand-in for a real part in a test: dummy, stub, fake, spy or mock | §5 |
| test double fidelity | How closely a test double behaves like the real part | §5; CL-02 |
| test environment management | Keeping test setups known, stable and recorded | CL-10 |
| test isolation | Tests that cannot affect one another | CL-28 |
| test oracle | Where the expected answer comes from; never the same computation the test checks | §4; CL-07 |
| test plan, test report | What will be tested and how; what happened | `docs/HARDWARE_TEST_PLAN.md` |
| test pyramid | Many fast unit tests, fewer integration tests, very few slow real-world runs | §2 |
| test-driven development (TDD) | Writing a failing test first ("red"), then the code that passes it ("green"), then tidying | §3 |
| testing the test code | Tests for the test tools themselves | §7; WORKFLOW.md §16 |
| time-boxing | A fixed time budget; when it runs out, stop and decide | §2; WORKFLOW.md §11, §17 |
| tooling hygiene | Careful habits with shells, scripts and git that avoid self-made errors | Playbook §4; WORKFLOW.md §3a |
| unit test | A test of one function or class on its own | `test/unit/` |
