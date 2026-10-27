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

5. **`pending` denies a save between brief and clear** (see bug 4 above; kept as
   its own number so the order below still reads).

6. **The `threshold` setting is ignored (confirmed 2026-10-27; resolved the
   same day, 0.10.1).** Cause: the settings.json entry lacked the `options`
   wrapper. The app passes `pluginConfigs.<key>.options` to the plugin, not
   the key's own fields, so `threshold` written straight under `auto-handoff`
   never reached `parseConfig`. The working entry is
   `"pluginConfigs": { "auto-handoff": { "options": { "threshold": 250000, "viewer": "" } } }`;
   the key for a `CLAUDE_CODE_PLUGIN_DIRS` plugin is the plain plugin name, no
   marketplace suffix. Verified in a fresh session: `/handoff-threshold` reports
   `250k (threshold in /config)`. The README now shows the entry, and the
   soft-ceiling clash noted below is handled: the warning is skipped when the
   threshold in force is at or below `softCeiling`. Original report follows.
   Otto set
   `pluginConfigs.auto-handoff.threshold: 250000` in `~/.claude/settings.json`,
   which is where the README's Configure section says settings live. Every log
   line today reads `threshold=160000`, and `/handoff-threshold` reports the
   source as "threshold in /config", meaning `parseConfig` in `hooks/config.ts`
   received no `threshold` in `options` and fell back to `DEFAULTS`. No chain
   override file exists under `~/.claude/state/auto-handoff/overrides`. The
   plugin is loaded through `CLAUDE_CODE_PLUGIN_DIRS` (a dev directory, plugin
   name `auto-handoff`), not from a marketplace; the likeliest cause is that the
   app keys `pluginConfigs` differently for a directory plugin, or passes no
   options to it at all. Two things to check: what `options` contains at
   `register()` for a directory plugin, and whether the key the app expects is
   `auto-handoff@<marketplace>` rather than `auto-handoff`. Note also that 250k
   equals the default `softCeiling`, so once the setting takes effect the soft
   warning and the handoff would fire at the same line; `softCeiling` should move
   up or the soft warning should be skipped when it is not below the threshold.

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
