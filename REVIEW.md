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

## Repeated past findings: always check

Seed rows from the September 2026 audit (`docs/audit-plan.md`, `docs/claims-evidence.md`); add one
here whenever the never-again ledger gains a row.

- A documented rate that the profile or timer does not produce (radar was documented at 16 Hz; the
  `frameCfg` period is 50 ms, so 20 Hz).
- Fusion described as IoU when the code associates by point-in-padded-box containment.
- A throughput number with no artefact behind it (YOLO "~35 Hz"; measured 22–27 Hz in-pipeline).
