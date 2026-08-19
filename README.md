# Battle Chronicles

Battles that produce a story instead of a casualty count. Ten moments fire from
CK3's combat phase events, apply temporary `advantage` modifiers to the
commander, and leave grudges and bonds behind them.

Design: `../../docs/specs/2026-08-12-battle-chronicles-design.md`. Specs, plans
and the API dumps all live in the tooling repo two levels above this one, so
every `docs/` path quoted anywhere in this repo is written relative to this
repo's root. Script comments that cite an API dump by bare filename — `.log`
names such as `effects.log:3507` — mean a file in `../../docs/api/1.19.0.6/`.
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
| `common/combat_phase_events/bc_commander_phase_events.txt` | 238 commander moments (7 shipped by plan 1, the rest from the content sessions) |
| `common/combat_phase_events/bc_knight_phase_events.txt` | 206 knight moments (3 shipped by plan 1) |
| `common/modifiers/bc_momentum_modifiers.txt` | the momentum ladder, 3 tiers each way |
| `common/scripted_effects/bc_scripted_effects.txt` | how momentum is applied, cleared, paced |
| `common/scripted_triggers/bc_scripted_triggers.txt` | every situation gate, one place, incl. `bc_battle_is_notable_trigger` |
| `common/script_values/bc_script_values.txt` | the four ratio thresholds |
| `common/game_rules/bc_game_rules.txt` | frequency, swing, aftermath |
| `common/on_action/zz_bc_on_actions.txt` | combat-end hooks, `events = { }` only. The `zz_` is a **failed** ordering experiment and is now arbitrary — see the file header |
| `common/character_memory_types/bc_memories.txt` | shieldbrother + the two battle memories |
| `events/bc_events.txt` | the five player-facing pop-ups, and nothing else |
| `events/bc_aftermath_events.txt` | the two hidden combat-end graders |

## Rules this mod lives by

- **Non-lethal.** No `death`, no maiming, anywhere. A setback is an advantage
  penalty and some stress.
- **Momentum never outlives its battle.** The combat-end graders
  `bc_chronicles.0091` and `.0092` strip it, before anything reads the scratch
  variables it was earned alongside.
- **Cooldowns are flags, never modifiers.** Modifiers are wiped at combat end, so
  a modifier-based lockout silently resets every battle.
- **Modifier names are prose.** The combat advantage breakdown prints them
  verbatim next to `Commander Martial Skill`.
- **Nothing is replaced.** Every file is new and additive.
- **State first, presentation second.** No gameplay effect ever sits inside a
  `send_interface_toast` — effects go in the `effect` block, the toast stays
  purely presentational. The *reason* originally given for this rule was wrong
  and is corrected below: a toast body **does** run for an AI subject. The rule
  stands anyway, because vanilla writes it this way and because a presentational
  block is the wrong place to hide state a future reader has to find.
- **A toast is addressed, not broadcast.** Knight chronicles are raised on the
  side commander, because nobody is playing a courtier knight.

## Where the rest of this lives

This README stayed at 522 KB for six days and could not be read whole by any
tool with a size limit, which is how 220 batches got written without anyone
re-reading the record. The long-form material now lives in the tooling repo
beside the specs it already cited, and each file is small enough to read:

| file | what | read it when |
|---|---|---|
| `../../docs/battle-chronicles/open-questions.md` | every mechanism no run has proven, with the marker/verdict tables | **before designing a run** - this is the agenda |
| `../../docs/battle-chronicles/runs.md` | all nine in-game runs, oldest first, and the three early review passes | before claiming anything is measured |
| `../../docs/battle-chronicles/batch-ledger.md` | the 2026-08-17 content session, 220 rows, split into six parts | before writing a new batch, to find what the axis already reads |
| `../../docs/battle-chronicles/phrase-ledger.md` | the spent-phrase index, formerly 72k comment lines inside the loc file | before writing prose, to avoid repeating a phrase |
| `../../docs/battle-chronicles/estructura.md` | diagrams of the mod's structure and how it hooks the game, in Spanish | to orient, or to hand the design to somebody else |
| `../../docs/battle-chronicles/history.md` | the chronological development account, lifted out of the workspace memory file | when the question is why something is the way it is |

**Last measured run: 2026-08-15.** Everything after it - the whole 08-17 content
session and the 08-18 batches - is static verification only. `validate --strict`
passing is not evidence a gate opens.

## The aftermath system

At combat end, `bc_chronicles.0091` (winner) and `.0092` (loser) grade the
battle. If it was notable, every surviving knight on the side gets a 40-year
memory of the province — `bc_battle_memory_won` or `bc_battle_memory_lost`. On
the winning side the commander also banks a notable win in
`var:bc_notable_battles`, and at three wins with `martial >= 14` and no nickname
yet, he is named on the field from a list of 27 AGOT nicknames.

One event per hook, not one shared event, because nothing in a combat-end event's
body can tell which hook invoked it. `is_combat_side_pursuing` is documented as
"is this side the winner", but every script use of it in vanilla and AGOT is
*during* combat; its value at combat end is unproven. A hook apiece makes the
outcome structural instead of inferred, and it is also what lets cleanup, grading
and scratch clearing run in a written, provable order within each one.

The grade must be **symmetric** — both sides must reach the same verdict, or one
side's knights remember a battle the other side's do not. Every row is either
combat-scoped (symmetric for free) or written as an explicit `OR` across
`enemy_side`.

Game rule `bc_aftermath`: `off` / `default` / `generous`. Generous donates a free
`calc_true_if` point, so one real row is enough. All three of the mod's game
rules default to their middle option.
