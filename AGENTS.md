# Hard rules for every agent and person working in this repo

**Owner: John. Applies to Claude, Codex and every other agent.** Adapted on November 23, 2026 from
the SaaS repo's AGENTS.md: the shop's rules (postage, payments, Next.js) do not apply here; these
are this system's equivalents. `CLAUDE.md` imports this file, so the same rules bind every session.
The reason behind each rule is in `docs/engineering-playbook.md` §8.

## ⛔ Nothing writes to a sensor or a board as part of a test

- **Never run `scripts/send_radar_config.sh`, a firmware flash, or anything else that writes to the
  radar, the camera or the Jetson's persistent state** while testing, debugging, reproducing a bug or
  "verifying the flow". "Test it", "finish the checklist" or "make sure it works" does **not**
  authorize it. Only John's explicit word, with John present, does, and the record names it.
- **Read-only is fine:** listing devices, reading `/dev/radar_*` streams, `ros2 topic hz`, reading
  the parameter dump, reading `src/cuas_fusion/config/radar_profile.cfg`.
- **A check that would need a write is reported as blocked**, never worked around.

## Models and calibration are never regenerated as a side effect

- A script that writes into `models/`, `calibration_frames/`, or the calibration files in
  `src/cuas_fusion/config/` (`camera_calibration*.yaml`, `extrinsics.yaml`) runs only on John's word.
  Every measured number in the docs depends on those files; replacing one silently invalidates the
  record.
- These paths are gitignored or irreplaceable: back them up outside the tree, with checksums, before
  any batch that could touch them (playbook §8 rule 1).

## Nothing leaves this machine during a test

- The Cursor-on-Target output (`cot_publisher_node`, UDP multicast) goes to ATAK clients on the
  network. A sim, replay or bench run keeps it local; if a run cannot, report the check as blocked.
- No credentials in code, launch files, logs, receipts or test fixtures. They come from a gitignored
  env file. Secrets are never printed; masked prefixes or fingerprints only.

## The decision path is the owner's

- Any threshold, gate, class policy or enum that changes what the system calls a threat, an
  intent, a geofence breach or a confirmed track is **"Needs John's decision"**, never assumed.
  `docs/HANDOFF.md` lists the decisions still open (D-0, D-1, D-2 …); do not build them silently.
- Every number that will be printed in the README, a résumé or an interview carries its conditions
  and date (sim or live, which sensor, board, model precision, clock source). An unmeasured number is
  worded as unmeasured.

## Git and GitHub

- Never push to `main`. Every change goes through a pull request; John merges.
- Never delete a branch, close a PR, rewrite history or rename the repo without John's go in this
  session, and never without a backup tag pushed first and the evidence written down.

## Browser windows on the second monitor (John, October 15, 2026)

Every Chrome window an agent opens (Playwright, a script, or by hand) opens on John's **second
monitor**, never the main one. Read the layout first if it may have changed:
`Add-Type -AssemblyName System.Windows.Forms; [System.Windows.Forms.Screen]::AllScreens`, and use the
screen whose `Primary` is False.
