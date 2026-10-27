# Tool-gate denials at the threshold leave the successor guessing

Raised 2026-10-27 by the Editor coordinator session after step 1.5 of the Editor
project ran through six handoffs in one morning. Plugin at `0b5fd84` (0.10.0),
threshold in force 160k, Fable 5.1, Windows. Log: `~/.claude/state/auto-handoff/auto-handoff.log`
between 07:30 and 08:20 UTC.

## What happened

Three child sessions of one coordinator hit the threshold while editing code.
Each time the `tool.call` gate (`hooks/register.tsx` around line 547) denied the
model's next batch of calls, then `turn.step` wrote the brief and cleared.

| Session | Denied calls | Result |
|---|---|---|
| `e701d99c` → `55cc534b` → `4f612415` (piece 2) | Read, then Write after the brief was written | Successor seeded at 58k, hit 160k again after 4.5 min, brief confidence Low. Stopped by the coordinator with no commits. |
| `203d7549` → `79823a8a` (piece 2a) | Write, PowerShell, Edit in one batch | Successor finished the work, two commits. |
| `5596fca1` → `3f94e102` (piece 2b) | Eight Edits in one batch | Brief: "edits returned errors, error text truncated, reason unknown", confidence Medium. Successor spent its first minutes grepping to learn which edits had landed; one function referenced a constant that was never added. It then finished, one commit. |

## The bugs

1. **The brief does not recognise the gate's own denials.** Haiku reads the denied
   calls as tool errors with truncated text and writes "file state is uncertain".
   The plugin knows exactly which calls it refused (it logs them), so the brief
   could list them under a heading like "Calls refused by the gate, never applied".
   The successor would then know which edits are missing instead of re-deriving
   the file state from grep. This is the cheapest fix and removes most of the
   damage seen today.

2. **The gate cuts across a multi-batch edit.** Piece 2b had applied the imports,
   constants and helpers of a change when the gate refused the batch holding the
   function bodies. The code was left inconsistent (`find_pacing` referenced a
   constant from the refused batch). The hard-ceiling path already lets
   `HANDOVER_TOOLS` (Read, Write, Edit, Glob, Grep, Skill) through; the threshold
   path refuses everything. Letting Edit and Write through within a small margin
   above the threshold, the way `OVERRIDE_MARGIN` works for the ceiling, would
   let the batch in flight land before the handoff.

3. **The denial message promises something that does not happen.** It says
   "the fresh session will redo this call". The fresh session reads the brief; it
   never sees the refused call's arguments. After a denial the model should be
   told to state, in its final text, which edits did not land, so the brief
   writer captures that (this also helps with bug 1 until it is fixed in code).

4. **`pending` denies a save between brief and clear.** In `55cc534b` a Write was
   refused 54 s after the brief was written and 20 s before seeding. Whatever the
   session tried to save in that window is gone. If this is by design, the
   message should say the brief is already written rather than the generic text.

5. **Threshold setting not in force (unverified cause).** `~/.claude/settings.json`
   has `pluginConfigs.auto-handoff.threshold: 250000`, but every log line today
   reads `threshold=160000` and `/handoff-threshold` reports the source as
   "threshold in /config". No chain override file exists under
   `~/.claude/state/auto-handoff`. Either `pluginConfigs` is not where the plugin
   reads from, or the default wins over it.

## Context that is not a bug, but shapes the fix

- The seed floor in this environment is about 58k tokens (system prompt, tool
  schemas, CLAUDE.md, memory). With a 160k threshold that leaves roughly 100k of
  working room; a code-editing session used it in 4.5 to 7 minutes. A successor
  that starts by reading the brief, the details file and the files the brief
  names is already near 80k before it edits anything. Piece 2 wedged this way:
  two hops, no commits, each brief thinner than the last.
- The sessions that recovered (2a, 2b) were the ones whose first commit had
  already landed before the gate fired. Encouraging a commit before the brief
  (a "commit what passes" step in the handoff instructions) would make the
  recovery rate less dependent on luck.

## Suggested order

Bug 1 first (brief lists refused calls), then bug 3 (message text), then bug 2
(let the batch in flight land). Bug 5 needs a look at `parseConfig` in
`hooks/config.ts`. Bug 4 is a message change if the behaviour is intended.
