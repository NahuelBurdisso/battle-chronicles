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
| `common/character_memory_types/bc_memories.txt` | the shieldbrother memory |
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

## Not yet verified in game

- Whether a `send_interface_toast` body runs at all for an AI character.
  `bc_knight_holds_the_breach` carries a `debug_log` inside its toast for exactly
  this: if `probe 'BC:'` returns fewer "toast body ran" than "fired
  bc_knight_holds_the_breach", the answer is no. Nothing depends on the outcome
  any more — this is to settle the rule, not to fix a bug.
- The memory: that it renders, and that `[shieldbrother.GetFullName]` resolves.
- The retuned weights.
