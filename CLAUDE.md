@AGENTS.md

@WORKFLOW.md

Before relying on an older handoff, plan, audit row or remembered finding, read the `PICK UP HERE`
block at the top of `docs/HANDOFF.md` and verify the current source and working diff. Old unchecked
items are historical evidence, not proof that the work remains undone. Never overwrite newer code
with an old audit copy.

The deep method (tiers, lifecycle, roles, harnesses, audit mode) is `docs/engineering-playbook.md`;
the review guide is `REVIEW.md`. Open them when the task needs them; they are not loaded here so
every session stays small.

## Keeping HANDOFF.md small

New entries go at the top, under `PICK UP HERE`, newest first. When the file passes about 100 KB,
move the oldest entries, unchanged, into `docs/state-history/` and keep the top block true.
