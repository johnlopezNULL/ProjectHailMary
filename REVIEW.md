# Review guide

Every reviewer reads this before a review: Opus reviewers, Codex, and the automated review on a PR.
AGENTS.md binds here too; WORKFLOW.md says where review sits. Adapted on November 23, 2026 from the
SaaS repo's REVIEW.md; the role-by-role lenses are in `docs/engineering-playbook.md` §4.

## What failure looks like in this repo

A sensor-to-decision system: radar bytes → detections → tracks → threat / intent / geofence →
operator view and Cursor-on-Target. A bug here is rarely a red test. Look hardest for:

- **Wrong decisions on real input:** a friendly escalated to THREAT, a threat never escalated, a
  geofence breach missed, an intent label that flips on noise.
- **Track identity:** IDs that swap when two targets cross, tracks that never die, a confirmed track
  from one clutter return, an association matrix off by one.
- **Fail-open on bad input:** a missing parameter, corrupt model or calibration file, malformed TLV
  frame or lost sensor that leaves a node publishing plausible data instead of last-known-good plus
  one log line.
- **Units, frames and time:** metres vs millimetres, degrees vs radians, `radar_frame` vs `base_link`,
  `use_sim_time` mismatches, stamps from the wrong clock, rates that no longer match the ICD.
- **The contract:** a message or topic change that misses a publisher, a subscriber, the ICD or the
  version (playbook §8 rule 4); a QoS mismatch that drops messages silently.
- **One home per constant:** a threshold, bound or enum duplicated in a second file or compared as a
  string in one place and an enum in another.
- **Bounds:** a fixed container (`FixedVector`, `FixedMap`) that can overflow or silently drop; a
  capacity in `constants.hpp` the code does not enforce.
- **Coding standard:** a JSF / MISRA rule from `docs/CODING_STANDARD.md` broken without a documented
  deviation (DEV-001 to DEV-006).
- **Docs that disagree with the code:** a rate, count, latency or claim in README / ICD /
  architecture / latency budget that the code or a measurement does not produce.
- **Hardware safety:** any path that writes to the sensor, the board, a model or calibration outside
  John's word (AGENTS.md).

## Lenses

Use the ones the diff calls for; `docs/review-playbook.md` has the full method.

- **Line by line:** force every error and edge path. Empty cloud, NaN, a single detection, 32 tracks.
- **Removed behaviour:** list every guarantee the old code made (validation, gating, clamping, error
  handling, a log line) and check each still holds. Diffs that move a responsibility are the most
  dangerous.
- **Cross-file tracing:** every caller and consumer of what changed; `/tracks` has ten consumers.
- **Tests:** would the test still pass with the fix removed? Does it test intended behaviour, or the
  current code? Was an expectation weakened without a written reason?

## Evidence

- Every finding cites `file:line` and a concrete failure scenario: input frame, state or parameters,
  then the wrong outcome.
- Back it with a failing test, a bag or sim reproduction, or a code path read end to end.
- Anything about real-sensor timing, serial behaviour or GPU throughput is "needs hardware check",
  never accepted on reasoning alone.
- Try to refute each candidate before reporting it. No scenario, no finding.

## Severity

- **Important:** a real bug or risk the change introduces or must fix before merge. Decision-path,
  contract, fail-open and hardware-safety issues are always Important.
- **Nit:** style, naming, small clarity. At most five per review.
- **Pre-existing:** a real bug the change did not cause. Report it, mark it, do not block on it.

When a brief asks for HIGH / MEDIUM / LOW, grade the Important findings that way.

## Paths to skip

`build/`, `install/`, `log/`, `models/`, `calibration_frames/`, `logs/`, `camera_backups/`,
`graphify-out/`, `launch/_harness_*`, `scripts/_*`, `docs/state-history/`.

## A reviewer never

- Edits code or commits.
- Runs anything that writes to a sensor or the board, sends CoT onto a network, or touches GitHub
  beyond reading.
- Reads an env or credentials file or prints a secret.

Reading files, `git diff`, `git show` and targeted `colcon test --packages-select cuas_fusion` runs
are fine.

## Output format

1. Findings ranked by severity, Important first. Each: `file:line`, the failure scenario, the
   smallest fix.
2. Pre-existing findings, the same way.
3. Nits, at most five.
4. One line: "No further findings." If nothing is wrong, say so plainly.

## Test tools and harnesses

A test tool (WORKFLOW.md section 16) gets one review round (WORKFLOW.md 3d). Extra rounds did not
find timing bugs on the earlier web-shop project; mechanical checks did (`docs/lessons-ledger.md`).
So do these six checks before the line-by-line read, and give each one's result (pass or fail) with
the command output. They cover the tools the brief names, also those under "Paths to skip"
(`launch/_harness_*`, `scripts/_*`).

- **Sort every sleep.** Run `git grep -n -E 'sleep|wait_for' -- <the tool's paths>` and put each hit
  in one class: (a) the interval of a poll with a time limit, (b) a pause with a comment giving its
  reason, (c) a finding. A pause placed after an action and before the read that decides the result
  is Important.
- **The slow run is there.** The self-test includes a run where everything the tool waits for takes
  at least 1.5 seconds to answer, and the output shows that run passing.
- **Nothing real in a self-test.** Run `git grep -n -E '239\.[0-9]+\.[0-9]+\.[0-9]+|/dev/(radar|video|tty)' -- <self-test and fixture paths>`
  and look for real hostnames too; a hit is acceptable only inside a comment or as an argument to a
  pure function.
- **Each test added with a fix fails without it.** The mutation list (WORKFLOW.md 3b) shows, for
  each such test, its result with the fix taken out; a missing result is Important.
- **Expected values do not reuse the computation under test.** Each one comes from hand
  arithmetic, a reference implementation, a recorded output of a trusted run, or the ICD, and the
  test says which.
- **Each pass line says what it read** (WORKFLOW.md 3c): the counts or fields, the command or read,
  and the allowed failures by exact text. A status word such as NOMINAL is fine inside such a line
  (for example the `status` field of `/health/status` with the topics and window read); an exit code
  or status word with nothing named beside it is Important.

Also confirm that the tool starts listening before it acts, that the fake-versus-real table covers
every judged input and output in every state, and that the tool grows the one the brief names and
reuses its lists. End the report by saying, by name, whether a second round is needed; only a
change to what the tool may write or send, or to when a hard-stop action may happen, needs one. A
wait, a count or a field name is settled by a self-test case instead.

## Repeated past findings: always check

Seed rows from the September 2026 audit (`docs/audit-plan.md`, `docs/claims-evidence.md`); add one
here whenever the never-again ledger gains a row.

- A documented rate that the profile or timer does not produce (radar was documented at 16 Hz; the
  `frameCfg` period is 50 ms, so 20 Hz).
- Fusion described as IoU when the code associates by point-in-padded-box containment.
- A throughput number with no artefact behind it (YOLO "~35 Hz"; measured 22–27 Hz in-pipeline).

Carried over from the earlier web-shop project (`docs/lessons-ledger.md`), November 23, 2026:

- A fixed sleep before the read that judges an action (`scripts/play_and_echo.sh` sleeps 4 s after
  starting the bag): wait, bounded, for the system's own signal (ledger CL-01).
- An observer attached after the event: a subscriber started after the bag or the pipeline began
  publishing, then judging "the first message"; every topic is Volatile (`docs/ICD.md`), so what came
  before is lost. Subscribe first, or judge from a node's own counters (CL-03).
- A fake faster or cleaner than the real sensor: no jitter, no dropped frames, instant startup, or
  a different frame convention (the sim scenes that used +X as forward, playbook §14). Check the
  fake-versus-real table (CL-02).
- A bare pass flag: "exits 0" or "health NOMINAL" with no fields named, or a script that prints
  WARNING and still exits 0 (`scripts/check_outputs.sh`) used as a check (CL-08).
