# Multi-angle bug-hunt playbook

Adapted on November 23, 2026 from the SaaS repo's `docs/review-playbook.md`. Why it exists: there, a
quick "is it healthy" pass (build green, tests green, endpoints answering) reported a healthy site,
and an 8-angle adversarial review of the same code then found 10 real findings, two of which could
take the whole site down. The same gap exists here: "builds with -Werror, the GoogleTest suite green,
sim smoke NOMINAL" says nothing about a track swap at a crossing or a node publishing after its
sensor went silent. REVIEW.md is the short form; this is the method.

## Why the quick pass misses bugs

Operational checks answer **"does it run right now?"** and exercise only the happy path:

- A clean build proves the code compiles, not that it survives a malformed TLV frame.
- Passing tests prove what the tests cover, nothing else.
- A sim smoke proves one scripted scene on one clock, not two targets crossing, clutter, or a lost
  sensor.
- A rate in the README proves someone typed it, not that the profile or the timer produces it.

None of these can surface: error paths never taken yet, parameter values not yet tried, states only
real hardware produces, or library behaviour you assumed instead of read.

## Why narrow lenses find them

1. **Narrow lenses beat one general read.** A reader whose only job is "list every guarantee the diff
   removed" enumerates what a general read glides past.
2. **Removed-behaviour audit:** diff the guarantees of the old code against the new (a gate, a
   clamp, a validation, a log line, a fail-closed branch). Diffs that move a responsibility, such as
   from a node into a pure class or from one node to another, are the most dangerous.
3. **Failure forcing, line by line:** for each call ask what happens if the input is empty, NaN, out
   of range, late, duplicated, or the file is missing.
4. **Cross-file invariant tracing:** who calls X, who consumes the topic, which launch file remaps
   it, which constant duplicates it.
5. **Docs-first verification:** every claim about ROS, Eigen, OpenCV, TensorRT or the IWR6843 is
   checked against the installed headers, docs or the device, never memory.
6. **Refutation pass:** every candidate needs a concrete failure scenario (frame / state /
   parameters → wrong outcome) and is tested against reality where possible: a unit test, a sim
   scene, a bag. A finding nobody can reproduce is killed.
7. **Independent redundancy:** finders with no shared state. The worst bugs are found by two or three
   independently; the unique ones come from the specialist lenses. Worth it before a demo, a bench
   run or a release, not for every small diff.

## How to run it (the steps)

**0. Scope.** A diff (`git diff <base>`), or paths (`src/cuas_fusion/src/fusion/`). For "review the
whole project", prefer the diff since the last known-good review, plus full files for anything on the
decision path. For a full sweep use the playbook's audit mode (`docs/engineering-playbook.md` §15).

**1. Operational layer first (minutes).** The gates of `docs/engineering-playbook.md` §6. This gives
the reviewers a known-good baseline. Do not stop here and call it healthy.

**2. Fan out finder lenses**, parallel read-only subagents, one lens each. Every finder must quote
exact `file:line`, cite the header or docs for any library claim, and state a concrete failure
scenario. No scenario, no finding.

- **Line-by-line correctness:** every changed line; force each error and edge path.
- **Removed behaviour:** every guarantee the old code made, checked in the new.
- **Cross-file tracer:** all callers and consumers of what changed; dropped callers; settings that
  override or cancel each other (launch remaps, parameter files, defaults in code).
- **Units, frames and time:** every conversion, every frame id, every stamp and clock source.
- **Contract:** messages, topics, QoS and rates against `docs/ICD.md`.
- **Efficiency and timing:** work per frame, allocations on the hot path, anything that can blow the
  latency budget (`docs/latency_budget.md`).
- **Coding standard:** `docs/CODING_STANDARD.md` rules and its deviations.
- **Altitude / architecture:** is the approach right; what gets expensive to change later (for
  example, what moving off ROS 2 will cost each node).

**3. Dedupe by root cause**, not by `file:line`.

**4. Verify or refute each candidate.** Accept if two independent finders agree or you confirm it
directly (read the path end to end, a failing unit test, a sim or bag reproduction). Anything about
real-sensor timing is "needs hardware check" until it is driven on the bench.

**5. Rank by impact and report** `file:line` + scenario + fix direction: unsafe or wrong decision on
real input > node crash or build broken on the target > wrong data on the bus > wrong operator
display > latency budget missed > polish.

**6. Write the fix plan as a handoff entry** (`docs/HANDOFF.md`): per item the problem, the exact
location, the fix, the trade-offs and the verification.

## Standing lessons

- "Build + tests green" is the start of a health check, never the conclusion.
- The most dangerous diffs move a responsibility: every guarantee of the old home must be re-checked
  at the new one.
- Every number in the docs is a claim; a reviewer asks for the command that produced it.
