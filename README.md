# Battle Chronicles

Battles that produce a story instead of a casualty count. Ten moments fire from
CK3's combat phase events, apply temporary `advantage` modifiers to the
commander, and leave grudges and bonds behind them.

Design: `docs/specs/2026-08-12-battle-chronicles-design.md`.
Every mechanic here was proven first by the `bc_spike` probe mod — see its
README for the 13-probe result table.

## Commands

- Deploy:   `python tools/ck3.py deploy battle_chronicles`
- Validate: `python tools/ck3.py validate battle_chronicles`
- Enable:   `python tools/ck3.py playset add battle_chronicles`
- Read results: `python tools/ck3.py probe 'bc_'`

Load after AGOT. Zero file collisions by construction — confirm with
`python tools/ck3.py index -q && python tools/ck3.py conflicts` (`conflicts`
reads the index, so a stale index can only ever report a pass).

## What is where

| Path | What |
|---|---|
| `common/combat_phase_events/bc_commander_phase_events.txt` | 7 commander moments |
| `common/combat_phase_events/bc_knight_phase_events.txt` | 3 knight moments |
| `common/modifiers/bc_momentum_modifiers.txt` | the momentum ladder, 3 tiers each way |
| `common/scripted_effects/bc_scripted_effects.txt` | how momentum is applied, cleared, paced |
| `common/scripted_triggers/bc_scripted_triggers.txt` | every situation gate, one place, incl. `bc_battle_is_notable_trigger` |
| `common/script_values/bc_script_values.txt` | the four ratio thresholds |
| `common/game_rules/bc_game_rules.txt` | frequency, swing, aftermath |
| `common/on_action/bc_on_actions.txt` | combat-end hooks, `events = { }` only |
| `common/character_memory_types/bc_memories.txt` | shieldbrother + the two battle memories |
| `events/bc_events.txt` | three pop-ups, cleanup, and the two aftermath graders |

## Rules this mod lives by

- **Non-lethal.** No `death`, no maiming, anywhere. A setback is an advantage
  penalty and some stress.
- **Momentum never outlives its battle.** `bc_chronicles.0090` strips it at
  combat end.
- **Cooldowns are flags, never modifiers.** Modifiers are wiped at combat end, so
  a modifier-based lockout silently resets every battle.
- **Modifier names are prose.** The combat advantage breakdown prints them
  verbatim next to `Commander Martial Skill`.
- **Nothing is replaced.** Every file is new and additive.
- **State first, presentation second.** No gameplay effect ever sits inside a
  `send_interface_toast`. A toast only reaches the player playing the scoped
  character, so anything nested in one is unreliable for the AI — and almost
  every commander and every knight in the world is AI.
- **A toast is addressed, not broadcast.** Knight chronicles are raised on the
  side commander, because nobody is playing a courtier knight.

## First live run (2026-08-12)

8 of 10 events fired. `probe 'BC:'` counts: van_breaks_through 16,
horse_killed 15, banner_falls 10, flank_collapses 7, ground_favours_us 5,
rally_the_line 5, shieldbrother_forged 3, lines_bogged_down 1, plus 40 cleanup
calls and one 0003 pop-up.

- **Resolved:** `bc_script_values.txt` works. Reading `side_strength` into a
  script value and multiplying is valid — `bc_flank_collapses` and
  `bc_van_breaks_through` both fired.
- **Two bugs found and fixed:** the opinion modifiers needed a *positive*
  `monthly_change` (a decay rate, not a signed drift), and `add_gold = -100` is
  rejected outright — now `remove_short_term_gold = 100`.
- **Never fired:** both knight events. Diagnosed and fixed — see below.

## Changes made after that run

- **Knight events were unreachable, for three compounding reasons.** The pool
  they compete in is `knight_none` at `base = 2000`, twice the commander pool's,
  so `base = 40` was ~2% of a roll before any gate applied. On top of that,
  `prowess <= 6` describes almost no knight — knights are *selected* for prowess.
  Now `base = 150`, `prowess >= 10` for the breach, and `craven OR prowess <= 8`
  for the disgrace, which is the better story regardless.
- **Every gameplay effect moved out of its toast.** They only reliably run for a
  character a player is playing, so as written the momentum system risked being
  player-only. `bc_lines_bogged_down` proved it: it hit the enemy commander from
  outside a toast and its own commander from inside one.
- **Knight toasts now address the side commander.** Raised on the knight, they
  reached nobody.
- **Weights retuned** against the observed distribution: `horse_killed` 60 → 40
  (no situational gate, was behaving like a second `none`), `lines_bogged_down`
  60 → 100 (terrain gate is narrow enough on its own), `van_breaks_through`'s
  martial bonus 40 → 25.
- **Shieldbrothers now leave a memory**, `bc_shieldbrother_memory`, forty years,
  one for each man naming the other. Opinion decays; this does not.

## Review pass, 2026-08-12 (four bugs, all found by reading)

- **The knight's opinion ran backwards.** `bc_knight_holds_the_breach` used
  `add_opinion = { target = liege }`, which raises *the knight's* opinion of his
  lord. "Stood Fast in Battle" is the lord's view of the knight. Now
  `reverse_add_opinion`, matching the grudge in `bc_knight_shames_himself`.
- **Two events skipped the tier gate.** `bc_flank_collapses` had no
  `NOT = { bc_is_faltering_trigger }` and `bc_van_breaks_through` had no
  `NOT = { bc_has_momentum_trigger }`. Tiers are separate modifier keys, so a
  commander could carry minor and major at once and sit at ±9 — off the end of
  the ladder everything else is tuned against.
- **`mountains` was in both terrain lists**, so mountain ground both favoured
  you and bogged you down, and the two events competed for the same roll. It is
  now only in the bogging list; hills and forest already carry "good defensive
  ground", and wetlands and jungle alone are close to nonexistent in Westeros.
- **Shieldbrother burned its cooldown on a no-op.** `is_valid` can only ask
  whether the side has two knights, not whether a second one is off cooldown, so
  the event could win a roll and find nobody to pair with — after already
  spending the knight's 15-day lockout and logging a fire. Nothing happens now
  until the partner exists.

## The aftermath system

At combat end, `bc_chronicles.0091` (winner) and `.0092` (loser) grade the
battle. If it was notable, every surviving knight on the side gets a 40-year
memory of the province — `bc_battle_memory_won` or `bc_battle_memory_lost`. On
the winning side the commander also banks a notable win in
`var:bc_notable_battles`, and at three wins with `martial >= 14` and no nickname
yet, he is named on the field from a list of 27 AGOT nicknames.

Two events rather than one because `bc_chronicles.0090` fires from both hooks and
nothing in its body can tell which called it. `is_combat_side_pursuing` is
documented as "is this side the winner", but every script use of it in vanilla
and AGOT is *during* combat; its value at combat end is unproven. Two events
makes the outcome structural instead of inferred.

The grade must be **symmetric** — both sides must reach the same verdict, or one
side's knights remember a battle the other side's do not. Every row is either
combat-scoped (symmetric for free) or written as an explicit `OR` across
`enemy_side`.

Game rule `bc_aftermath`: `off` / `default` / `generous`. Generous donates a free
`calc_true_if` point, so one real row is enough. All three of the mod's game
rules default to their middle option.

## Calibration run, 2026-08-13 (39 combat ends, both sides logged)

The grade's thresholds were seeded from vanilla's own numbers and then measured
with temporary ladder markers. Vanilla's numbers turned out to be the wrong
*shape*, not merely the wrong size — **three of the five rows never fired once**:

| row | ladder | verdict |
|---|---|---|
| size | <5k 26 / 5-10k 8 / 10-20k 5 / 20k+ **0** | 20k and 30k were dead. Now 10k / 20k. |
| blood | <25 3 / 25-50 29 / 50-75 7 / 75+ **0** | 75 was dead. Now 50 (top 18%). |
| odds | even 9 / 1.2x 9 / 1.5x 11 / 2x 10 | 1.5x fired on **54%** and was carrying the whole trigger. Now 2x (top 26%). |
| warscore | <10 30 / 10-25 3 / 25-50 2 / 50+ 4 | 25 already sat at the top 15%. Kept. |

The three ratio rows are scale-free and will hold as hosts grow. Size is the only
absolute, which is why both rungs are set above the sample rather than fitted to
it — 20k is reserved for late-campaign hosts and is meant to age into relevance.
Re-derive by restoring the ladders from git history, not by guessing.

- **Resolved:** `side_max_army_size` works on the LEFT of a comparison. It had
  zero script precedent anywhere in vanilla, AGOT or the playset. A control
  ladder against plain constants left its `else` branch, and the odds ladder
  spread across all four buckets instead of collapsing to one.
- **Resolved:** a `send_interface_toast` body **does** run for an AI character —
  `fired bc_knight_holds_the_breach` 14 == `toast body ran` 14.
- **Resolved:** all 11 phase events now fire, including
  `bc_knight_shames_himself` (11×), which never fired before the retune.
- **Resolved:** symmetry holds empirically. Every ladder matches rung-for-rung
  between `.0091` and `.0092` across all 39 battles, and `win notable` 5 ==
  `lose notable` 5. Only the per-side host ladder differs, which is correct.
- **Resolved:** zero mod-attributable lines in `error.log`.

## Confirmation run, 2026-08-13 (Generous, 5 combat ends)

A second, short run on `bc_aftermath_generous` to settle the two things the
calibration run could not.

- **Resolved: two knights on the same side both get a memory of the same
  battle.** Checked by eye in the memories window on two different knights after
  a player-fought battle; both name the same province. This was the one thing in
  the design no log could answer, and it is the load-bearing proof that
  `bc_battle_memory_effect` is iterator-safe — the `random_memory` +
  `NOT = { has_variable = battle_location }` idiom does not let the second
  knight's `set_variable` land on the first knight's memory.
- **Resolved: `bc_aftermath_generous` executes.** 3 of 5 battles graded notable
  against a Default rate that is now well under the 13% the calibration run
  measured on looser thresholds.
- **Symmetry held again:** win notable 3 == lose notable 3, win not-notable 2 ==
  lose not-notable 2. All 5 battles, both sides, same verdict.
- Zero mod-attributable lines in `error.log`.

## Not yet verified in game

- The shieldbrother memory rendering, and that `[shieldbrother.GetFullName]`
  resolves.
- The nickname award. It needs one commander to bank three notable wins with
  `martial >= 14` and no nickname yet, which no run so far has been long enough
  to produce. `"BC: nickname awarded"` is a permanent marker, so `probe 'BC:'`
  will report it the first time it happens.
