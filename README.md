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

Load after AGOT and after Battlefield Duel+. Zero file collisions by
construction — confirm with `python tools/ck3.py conflicts`.

## What is where

| Path | What |
|---|---|
| `common/combat_phase_events/bc_commander_phase_events.txt` | 7 commander moments |
| `common/combat_phase_events/bc_knight_phase_events.txt` | 3 knight moments |
| `common/modifiers/bc_momentum_modifiers.txt` | the momentum ladder, 3 tiers each way |
| `common/scripted_effects/bc_scripted_effects.txt` | how momentum is applied, cleared, paced |
| `common/scripted_triggers/bc_scripted_triggers.txt` | every situation gate, one place |
| `common/game_rules/bc_game_rules.txt` | frequency and swing |
| `common/on_action/bc_on_actions.txt` | combat-end hook, `events = { }` only |
| `events/bc_events.txt` | the three pop-ups and the hidden cleanup |

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
- **Never fired:** both knight events. Gates are narrow (prowess >= 12 or <= 6,
  *and* the side losing, at base 40 against `commander_none` at 1000).
  `bc_shieldbrother_forged` fires from the same file, so the hook is fine.

## Not yet verified in game

- Whether vanilla's own `scope:combat_side.enemy_side.vastly_outnumbered_combat_side_threshold`
  should replace `bc_outnumbered_threshold_value`. It exists and does the same
  job; the hand-rolled value is proven, so this is a simplification, not a fix.
- The weights. `base = 60` across seven commander events is a guess aimed at
  15-20% of commander rolls. `bc_frequency` exists so this is retunable without
  a script edit.
- `bc_shieldbrother_forged` grants opinion only. The design also calls for a
  character memory, which needs a `common/character_memory_types/` entry and has
  not been probed.
