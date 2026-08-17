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
| `common/combat_phase_events/bc_commander_phase_events.txt` | 39 commander moments (7 shipped by plan 1, the rest from the content sessions) |
| `common/combat_phase_events/bc_knight_phase_events.txt` | 28 knight moments (3 shipped by plan 1) |
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
- **Every gameplay effect moved out of its toast.** The stated reason — that a
  toast body only runs for a character a player is playing — was a *guess* from
  the documentation wording, and the 2026-08-13 run disproved it (see below).
  The move was still correct; the justification was not.
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

## Forced-nickname run, 2026-08-14 (expansion plan 1, task 2)

The nickname reward path had never executed, so plan 1 forced it before plan 1
task 4 refactors it. The gate was temporarily reduced to `has_any_nickname = no`
— dropping `exists = var:bc_notable_battles`, `var:bc_notable_battles >= 3` and
`martial >= 14` — and restored afterwards with `git checkout --`. No source
change survives this run.

- **Resolved:** `bc_aftermath_nickname_effect` executes. `BC: nickname awarded`
  appeared 3 times over 4 notable wins, and the toast rendered in game. The name
  drawn was `nick_agot_greycloak`, so the `random_list` of 27 and AGOT's nickname
  keys both resolve.
- **The toast is addressed correctly.** It reached the player.
- **The toast text is generic and that is a real gap.** All 27 nicknames share one
  `bc_aftermath_nickname_toast_desc`, which never names the nickname it just gave
  — the player sees "men have started calling him something" and has to open the
  character to learn what. Noted for plan 6.
- **Error sweep clean.** `probe "bc_"` returned 15 groups, all whitelisted: the
  mod's own `BC:` markers, the launcher's `BC Spike|...|Disabled` line, and two
  engine load lines (`NAMESPACE > 'bc_chronicles' is set to #4180000`,
  `Loaded [6] events from 'events/bc_events.txt'`). Zero lines in `error.log`.
- **A wasted run came first, and the cause is worth writing down.** The mod had
  fallen out of `dlc_load.json` since 2026-08-12, so the first attempt produced
  zero `BC:` lines from a full battle. `deploy` writes the pointer `.mod` only; it
  does not enable the mod. Check `debug.log` for
  `Battle Chronicles|mod/battle_chronicles.mod|Enabled` before trusting any run.

## DIAG run, 2026-08-14 (expansion plan 1, task 15)

Five open questions were instrumented with pure `debug_log` markers and answered
in one session. 137 combat ends per grader. The DIAG markers are temporary and
come out in task 16; the answers are permanent and live here.

- **Q1 — commander trait density.** n=267 side commanders: 0 traits 153, 1 trait
  110, 2 traits 3, 3+ traits 1. `number_of_commander_traits < 3` passes 99.6% of
  the time, so it is not a gate in practice. Keep it; it costs nothing and it
  covers the one case in 267.
- **Q2 — `scope = combat` events work; the combat links do not.**
  `bc_chronicles.0900` — the probe event, since retired — fired **151** times, so a
  `scope = combat` event dispatched from `on_combat_start` is valid even though
  nothing in vanilla or AGOT declares one. But `combat_attacker.side_commander` and
  `combat_defender.side_commander` each resolved **zero** times from inside it, and
  `?=` swallowed both failures in silence.
  **This conclusion was later superseded — see the 2026-08-14 run.** The two links
  were probably never the problem: no commander is assigned to a combat side at
  `on_combat_start` at all, so *every* route out of that hook finds nobody. The
  hook is now unclaimed and the opening runs from inside the battle.
- **Q3 — battle size.** Per side: 5k+ 46, 3k 42, 2k 31, 1k 14, under-1k 4. The
  invented 2,000 boundary in `bc_battle_open_effect` splits the low band 73/18
  instead of leaving one undivided lump. Keep it.
- **Q4 — a toast body DOES run for an AI subject.** Split by subject: human 0,
  AI 117, and the marker *inside* the toast ran 117 times. **B == H + A**, so
  nesting is not what suppresses anything. This retires the mod's original stated
  reason for keeping effects out of toasts — the practice stays, the explanation
  was wrong. The earlier probe could not settle this because it did not split by
  who was playing the subject.
- **Q5 — terrain census.** rough 58, forest 32, winter 20, open 19, jungle 8,
  **desert 0**, and `NONE MATCHED` **0** — the `agot_is_<x>_terrain` ladder is
  exhaustive over everything that actually occurred. Desert's zero is recorded
  rather than acted on; the arm stays.
- **Other counts:** `win notable` 79 / `lose notable` 79 and `win not notable` 58
  / `lose not notable` 58, perfectly symmetric as they must be.
  `BC: nickname awarded` 5, this time under the real gate.

### The run also exposed a defect that made the whole ledger dead

`bc_record_moment_effect` declared `SWING` in its signature and read it nowhere.
CK3 does not ignore an unused parameter — it **refuses to compile the effect**:

```
[E] Compiling source for bc_record_moment_effect failed for unknown arguments: SWING
    bc_ledger_effects.txt line: 119
```

thirteen times at load, and **zero** `BC: moment` lines all session. All 18 call
sites were silently dead. Nothing else broke: no toast, no event, no call site
complained. The fix reads the parameter with a two-arm log rather than deleting
it, so plan 6's wound gate keeps the `SWING = down` hook it is written against.

Three follow-on lines were `Variable 'X' is used but is never set` for
`bc_battle_style`, `bc_battle_tier` and `bc_battle_wounded` (twice each). The
first two resolve once the seam compiles. The third is genuine and stays until
plan 6 — and it corrects a claim the plan made in passing: `remove_variable` on a
never-set variable **does** log.

## Checkpoint run, 2026-08-14 (plans 2, 3 and 4 in one session)

Three deferred runtime checkpoints — plan 2 task 12, plan 3 task 10, plan 4 task
11 — run together on one debug session rather than three. 37 notable losses, 707
moments recorded.

**Error sweep clean.** `error.log` holds exactly one distinct `bc_` group:
`Variable 'bc_battle_wounded' is used but is never set`, twice, which is the
known plan-6 debt, since retired — `bc_wound_effect` writes the variable, and the
error sweep under "Confirmation run, 2026-08-15" below found no `bc_` group at
all. Every `debug.log` group is either a mod marker or one of the two engine
load lines plan 1 carved out. **Zero missing-variable reads** for anything plans
2, 3 or 4 introduced — which is the real result, because all three read
variables that may not exist and rely on nested guards rather than
short-circuiting.

### Plan 2 — the ledger

- `BC: size` covered 3 of 4 bands (size 2 ×120, size 3 ×96, size 1 ×40).
- `BC: verb` hit **all four** verbs: shieldwall 57, pursuit 42, bloodied 52,
  spent horse 40. The verb table is exercised end to end.
- `BC: chronicle mark commander` ×139.
- **`BC: floor admitted` never fired.** Per plan 2 task 12 step 6 this is
  *inconclusive, not a failure* — that row only applies when the mark marker is
  absent, and it fired 139 times. Retry recipe: two battles back to back with the
  same commander. **The floor is untested, not broken.** Every other empty probe
  in plan 2 means a defect; this one does not, and it is the result a later
  reader is most likely to misremember.

Two structural notes from plan 2 that belong with these counts:

- **The battle-size boundary is measured, not invented.** Q3's census (per side:
  5k+ 46, 3k 42, 2k 31, 1k 14, under-1k 4) is what the 2,000 rung in
  `bc_battle_open_effect` is set against — it splits the low band 73/18 rather
  than leaving one lump. Plan 6's wound gate inherits it.
- **All four verbs are reachable from the shipped call sites**, so a missing verb
  after several varied battles is a defect to investigate, not a shrug. What is
  unreachable is three `(KIND, SWING)` *pairs* out of twelve — `wild`/`up`,
  `entrenched`/`down`, `ordered`/`down` — which is why only the `ordered` row of
  the verb ladder tests `SWING` at all.
- **`bc_mark_chronicle_effect` and `bc_chronicle_ready_trigger` are knight-only
  now.** The commander pool got its own pair, so the two pacing budgets stopped
  competing: a commander moment no longer spends a knight's slot.
- **Known pre-existing hazard, not fixed and not worsened.** A pop-up option
  answered *after* combat ends applies its momentum modifier with a fresh 20-day
  duration, and `bc_cleanup_effect` has already run, so nothing strips it until
  it expires. This predates plans 1 and 2 and affects `bc_chronicles.0001`–`.0003`;
  the 60-day pop-up cooldown keeps the window narrow. The family-2 modifiers are
  immune only as a side effect of the `exists = var:bc_battle_size` guard, which
  is there for correctness of the read — **do not remove that guard on the
  grounds that the variable is always written, because at that call site it is
  not.**

### Plan 3 — style and terrain reputation

| | reset | streak | award |
|---|---|---|---|
| style | 67 | 7 | **1** |
| terrain | 69 | 8 | **8** |

Terrain awarded across 3 distinct argument hashes.
**No `BC: terrain NONE MATCHED`** — the terrain ladder is still exhaustive under
the reputation caller, matching Q5. **No `BC: award blocked` of any kind**, which
confirms Q1's estimate that the trait-count budget gate passes ~99.6% of the time
and shows style and terrain never collided over the same battle.

### Plan 4 — the nemesis

`BC: nemesis new` ×32, `escalate` ×6, `held` ×1, `rung grudge` ×26, `rung rival`
×12.

- **The accounting is exact.** At the first snapshot new 30 + escalate 6 + held 1
  = 37 = exactly the 37 `BC: lose notable`. Nothing was swallowed by the entry
  point's nested guards on `bc_battle_foe` existence and liveness.
- **`rung rival` (12) exceeding `escalate` (6) proves the wipe branch.** A first
  loss can only land on the rival rung if `bc_rivalry_step_value` returned 2, so
  `scope:wipe` resolves inside the script value — which had no precedent.
- **The descent path is unverified.** `decrement`, `cleared`, `wipe` and
  `rule off` were all absent, for the ordinary reason that nobody beat their
  nemesis in one session. All four are permanent markers and will report
  themselves.

### Bonus: plan 1's last open done-when closed

`BC: moment decisive` fired once. It had been 0 across 707 moments in the DIAG
run, and it was the single unmet criterion left on plan 1.

## Sweep run, 2026-08-14 (plan 5 task 11 and plan 6 task 6 in one session)

Counts below are the whole `debug.log` — 5,623 `BC:` hits over 96 distinct
markers, 169 combat ends per side, 137 notable wins and 137 notable losses. Three
results, two of them failures, and both failures were diagnosable from the marker
counts alone.

### The eve never reached a single commander, and the ratio said why

- `BC: eve fired` **68**, `BC: eve side` **136**, `BC: eve commander` **0**.

136 is exactly twice 68, so the `scope = combat` event dispatched and
`every_combat_side` visited both sides of every one of those 68 combats. Both
links worked. `side_commander` then produced nobody, 136 times out of 136.

That is not a broken scope link. The same `side_commander` reached a commander
**385** times in the same log through `BC: chronicle mark commander`. The
difference is *when*: **no commander is assigned to a combat side at
`on_combat_start`.** This also retires the DIAG Q2 conclusion above —
`combat_attacker` / `combat_defender` were very likely never the problem, there
was simply nobody there to find.

There is no other pre-battle hook in the on_action dump, so no routing fixes it.
The opening moved **into** the battle: `bc_eve_offer_effect` is now called from
the end of `bc_battle_open_effect`, which already runs on the side commander and
already holds the foe and the size, and gates itself to the first moment with
`NOT = { exists = var:bc_battle_moments }`. `on_combat_start` is left unclaimed
with the measurement written into the on_action file.

**The general lesson, which cost two runs: a `?=` block that never runs is
indistinguishable from one that had nothing to do.** Only the marker inside it
told the difference.

### The dead-knight toast was watching the wrong side

- `BC: dead knights` **0**, across 274 notable grader runs.

Vanilla kills knights on **one** side and not from the hook we were on.
`combat_event.1001` walks the winner at AGOT `combat_events.txt:846` only to write
battle history; it does its killing at `:849`, inside
`enemy_side = { every_side_knight = { ... } }`. So the dead knights of a battle
are always the **loser's**, and they are made dead on the **winner's** hook — a
place `bc_chronicles.0092` can never be ordered after.

The toast now lives once, in `.0091`, reaching across to `enemy_side` and
addressing the loser's commander. That requires our grader to dispatch after
`combat_event.1001`, which is why the on_action file is now
`zz_bc_on_actions.txt`: the merged `events` list is ordered by the same
alphabetical filename read order that decides an `effect` fight, and `bc_` sorted
before `combat_`.

### Family 3 is shipped but unmeasured

- `BC: moment decisive` at its shipped line **0**. `BC: wound` **0**.

The single `BC: moment decisive` in the log is at `bc_ledger_effects.txt:258`,
which is a **previous build** — the current one logs it from inside
`bc_record_effective_tier_effect`, at a line that has moved since. So no decisive
moment occurred at all during this run and the wound was never reached. This is
not evidence against the wound; it is evidence that `decisive` needs
`var:bc_battle_moments >= 5`, i.e. a long battle, and none happened. Plan 6's
firing rate remains unknown.

### Closed by this run

- **`BC: floor admitted` fired 5 times.** It was 0 in the checkpoint run and
  recorded there as *inconclusive, not a failure*. The cooldown floor is now
  exercised.

## Confirmation run, 2026-08-15 (logs cleared first)

Both 2026-08-14 fixes retested on a clean log. 40 combat ends per side, 28
notable wins, 28 notable losses, 167 moments. **Error sweep clean — no `bc_`
group in `error.log` at all**, which also retires the `bc_battle_wounded` debt
for good.

### The opening works, end to end

```
1  bc_eve_effects.txt:72 (bc_eve_offer_effect):          BC: eve offered
1  bc_eve_events.txt:70  (bc_chronicles.0011:immediate): BC: eve popup 0011
```

Gate passed, event raised, event opened — and **the pop-up named the enemy
commander**, confirmed by eye. That closes the two things no static check could
reach: `scope:bc_eve_foe` survives the `trigger_event`, and
`bc_eve_facing_nemesis_trigger` picks the nemesis branch correctly. The move off
`on_combat_start` is done.

### The dead-knight toast failed again, and the single marker failed again

`BC: dead knights` **0** over 28 notable wins, after the `enemy_side` rework and
the `zz_` rename.

**The block had one marker, inside the innermost `?=` — the same trap that hid
the eve's failure for two runs.** A zero from it cannot say which of three
different failures happened, and they need three different fixes. The block now
carries a marker per step (`BC: dk reached enemy side`, `BC: dk found dead`,
`BC: dead knights`); the next run names the failing step instead of returning
another uninformative zero. The scope path itself is not in doubt — vanilla does
`enemy_side = { every_side_knight = { ... } }` at `combat_events.txt:850`.

**The lesson is now three runs old and should be treated as a rule: a marker
inside a `?=` or a nested `if` only proves the innermost step. Mark every step
whose failure would need a different fix.**

### The decisive rung was measured too high, and has been lowered

`BC: moment decisive` **0** again, so `BC: wound` was **0** for the
uninformative reason a second time. But this run had a clean log and gave the
number that explains it:

| | |
|---|---|
| moments recorded | 167 |
| combat ends | ~40 |
| **moments per battle** | **~4.2** |
| old decisive floor | `var:bc_battle_moments >= 5` |

**The floor sat above the mean battle length.** Across both runs — 874 moments —
`decisive` fired exactly once, ever. The rung was not rare, it was very nearly
unreachable, and since family 3's wound hangs off that arm alone it could never
be measured at all.

Lowered to `>= 4`, so the ladder is now minor 1-2, major 3, decisive 4+. **The
major band is one moment wide and that is the part most likely to need a second
look.** Re-measure before moving either number again.

## Tiered-marker run, 2026-08-15 (logs cleared, both fixes measured)

Third clean-log run. 44 combat ends, 36 notable wins, 36 notable losses. Battle
sizes 1/2/3 = 17/85/131. Moments: minor 187, major 45, decisive 1 = **233, or
5.3 per battle** — up from 4.2, so the mean moves between runs and any floor set
against it is soft.

### The opening is closed

`BC: eve offered` 2 → `BC: eve popup 0011` 2. Second clean run in a row with a
1:1 ratio. Nothing outstanding.

### The `zz_` filename premise is disproven

```
33  BC: dk reached enemy side
 0  BC: dk found dead
 0  BC: dead knights
```

Row one of the three-way table the block was carrying: **`enemy_side` steps
correctly and `combat_event.1001` has not killed anyone yet when we look.** The
per-step markers paid for themselves on their first run.

**Alphabetical filename order does not order the merged `events = { }` list.** It
decides which `effect` block wins when two files claim the same on_action, and
that is all. `zz_bc_on_actions.txt` sorts last among filenames and still ran
before AGOT's 1001. *No prefix can fix this — do not try another one.*

The read is now deferred a day into `bc_chronicles.0093`, raised from `.0091`
inside the same notability gate, which steps outside the dispatch entirely. The
one thing that cannot be settled statically is whether the combat side object
survives the day; `BC: dk deferred` (call site) paired with
`BC: dk deferred fired` (event) answers it. First counting while the second is
absent means re-hang the read on the commander character instead of the side.

### The decisive floor was the wrong lever

`BC: moment decisive` **1**, and the logged line was checked against that run's
build and matched the `debug_log` inside `bc_record_effective_tier_effect`, so it
is a genuine clear of the new `>= 4` floor rather than a stale hit. The number
that line had then is not recorded here on purpose: it would be a claim about a
build this file can no longer point at. But the interesting number is upstream:

`BC: fired bc_flank_collapses` **10**, and both `TIER = decisive` call sites pass
identical arguments, so they share one arg-hash — which totals 10. So
`bc_chronicles.0001.c` was chosen **zero** times and every decisive *request* in
the run came from the flank-collapse phase event.

| where those 10 landed | count | means |
|---|---|---|
| `:311` minor | 7 | ledger held 1-2 moments at call time |
| `:307` major | 2 | ledger held exactly 3 |
| `:285` decisive | 1 | ledger held 4+ |

`5 → 4` was directionally right (this run scores 0 at `>= 5`), but the dominant
constraint is not the floor: **only 10 of 233 moments ever ask for `decisive`,
and 7 of those ask too early.** `bc_flank_collapses` fires near the start of a
battle. Leave the floor at 4 and re-measure before touching it again.

### The wound's zero was starvation, not a bug

`bc_wound_effect` had exactly one call site — the last line of the decisive arm.
Decisive resolved once, so **the wound had one opportunity in the entire run and
missed it.** Adding markers to it would still have produced a one-sample log.

It is now called from the **major arm as well**, which resolved 45 times on the
same run. Its own `var:bc_battle_wounded` guard still caps it at one wound per
battle, so this widens the sample without widening the outcome. A
`BC: wound gates passed` marker now splits the five-row outer limit from the
nested `var:bc_battle_size >= 3` read, which are the two candidates with
different fixes.

## Content session, 2026-08-17 (no run — static verification only)

Twenty-seven batches, one per row of the table below, none of which needed a
runtime answer. The measured problem they
answer is one number: **ten moment templates produced 233 firings in 44
battles**, so a player who fights one war reads the same ten sentences over and
over. Design: `../../docs/specs/2026-08-17-battle-chronicles-cases-design.md`.

**The rate limiter is the cooldown, not the weights.**
`bc_chronicle_ready_commander_trigger` fails while `bc_chronicle_recent` is
live, so a blocked roll is not a Battle Chronicles roll at all — it goes to
vanilla's `commander_none`. A template that fires in a slot an old one already
owned therefore buys variety at no rate cost. The honest half: an even battle on
open ground with an unremarkable commander used to match nothing, and now does.

| batch | what | count |
|---|---|---|
| commander, the second ten | ground bands, war length, host size, casualties, kin, age, the liege | 10 templates, base 30 |
| commander, character | wrathful, patient, drunkard, zealous, craven, strategist — the `has_trait` axis, which every earlier template used only as a `chance` modifier | 6 templates, base 30 |
| knight, the second three | the WINNING side (every shipped knight case gated on losing), outnumbering, ground | 3 templates, base 80 |
| knight, the third three | winter ground, age, kin — asked about the knight | 3 templates, base 80 |
| eve | three contextual options: horse country (`f`, the only option applying two modifiers and the only one applying a negative), kin (`g`) and heathen (`h`, the only option paid for in piety) | 8 options total |
| prose pools | bond clause 5→15 arms; **scale clause 3→9 arms**, the most-read prose in the mod since it closes every moment toast; eve foe 3 lines per value; eve ground 2 per band; memories 3 per perspective; nicknames re-cut into six style ladders over 35 keys | — |
| per-moment ground clauses | the spike had been the only one for two months; the other nine moments now each name their band. **All ten shipped moments name their ground.** Plan B tasks 6-14's *ground* half is done — the *core* sets, which replace a literal rather than following it, still wait on a run | 9 sets, 63 keys |
| commander, the last two bands | rough (paired with losing, so it answers `bc_banner_falls` instead of nesting inside `bc_too_wide_to_see`) and jungle (the band alone), plus the two missing combat-side band wrappers | 2 templates, base 30 and 40 |
| knight, the fourth three | the battle itself, read from inside the ranks — casualties, war length, host size. All three triggers were written for commander cases and had never been asked from a knight's root | 3 templates, base 80 |
| commander, who is across the field | the bond ladder had four answers and only *kin* had ever been a reason for a moment. Nemesis becomes a **gate** for the first time (it was a ×1.5 `chance` row on nineteen templates and a gate on none); heathen and countryman get the two new foe triggers | 3 templates, base 30 |
| knight, the trait batch | twelve knight cases and not one gated on temperament — traits appeared eleven times as `chance` rows and never as a reason. Arrogant, patient, greedy. **`bc_knight_walks_them_back` is the mod's first `ordered`/`down`**, the last unused cell in the seam | 3 templates, base 80 |
| the nemesis memory | the rivalry ladder's top rung left **nothing permanent** — `set_relation_nemesis` can be cleared, and clearing it erased every trace of the three fields. A memory type on the shieldbrother pattern, written on the loss that makes the count exactly three. The loser carries it; the winner does not | 1 type, 9 keys |
| knight, the ground batch | the knight side read two of six bands (open, winter) while the commander side reads all six. Rough (+ prowess, because 58 of 137), forest (+ winning, the mod's **second `ordered`/`down`**) and jungle alone. **No desert case**: it scored 0 of 137 and a template nothing reaches is not content | 3 templates, base 80 |
| shared ground clause | the ten Plan B moments each name their band; the **twenty-six toasts written since then named nothing underfoot**. One pooled set, two strings per band, appended ahead of the bond clause. Not called by the ten (they have per-moment sets) nor by the seven toasts gated on a band (their literal is the ground) | 13 keys, 26 toasts |
| pop-up contextual options | the three clickable moments had 3, 3 and 2 answers and had not been touched since plan 1. Zealous and purse on the banner, brave on the rally, greedy and `martial >= 14` on the van. **A pop-up can only ask about the man** — it is a `character_event` and `scope:combat_side` does not cross `trigger_event` | 5 options, 8 → 13 |
| the fourth pop-up | **three of thirty-one commander templates had ever asked the player anything**, and they were the same three since plan 1. `bc_flank_collapses` grows a pop-up branch — shape copied verbatim from `bc_banner_falls`, `else` arm the unchanged original, so AI commanders and players inside the 60-day cooldown see no change. **The only pop-up in the mod where every answer is a bad answer**: the choice is between a `minor` rung bought with stress, horses or nothing, and it is chosen over a kin pop-up because a `character_event` cannot reach the man across the field | 1 event, 4 options |
| commander, what he has to lose | twenty-eight commander templates and **not one read his station** — a king and a landless sworn sword got the same chronicle. One question, three answers: a crown on a *losing* field (narrowed so it does not own a player-king's chronicle), `is_landed = no`, and close family among his **own** knights. The third is the first gate in the mod to ask *who* the side's knights are; `bc_shieldbrother_forged` only ever counted them | 3 templates, base 40, 30, 40 |
| knight, the body batch | twenty-one templates read prowess eleven times and **never once asked what shape the man is in** — prowess is a number about how well he fights, not what he is fighting with. The `wounded_*` ladder while losing, `maimed`/`one_legged` with no side row at all, `one_eyed`/`disfigured` while winning. The health traits are the one axis the whole mod had never read, and a battlefield is where vanilla hands them out | 3 templates, base 80 |
| the eve's winter option | `d` took rough/forest/jungle and `f` took open/desert; **winter was the one band with no option of its own**, and deliberately so — its paragraph is neither a thing to hold nor a road to ride down. So `i` is not about the ground. It is **the only option in the mod that costs gold and the only one with a resource gate**: stress, prestige and piety are as available to a landless knight as to a king, and coin is not | 1 option, 8 → 9 |
| what he is already carrying | **the mod writes stress thirty-four times and had never read it back** — eleven `add_stress` in each phase event file, nine across the pop-ups, three in the eve, and `stress_level` in no gate anywhere. Every one of those writes says a battle costs a man something and not one of them was allowed to matter afterwards. Stress is not temperament: `craven` is a man who has always been like this, level 2 is a man who was fine in the spring. Commander misreading it while losing (`measured`/`down`, which grants nothing, and that is the point), commander who cannot stop while winning, and one knight case — a knight has no army, so he has only one answer | 3 templates, base 40, 30, 80 |
| the first knight pop-up | **four pop-ups shipped and all four assume the player is the man giving orders** — in AGOT he is very often somebody's sworn sword instead, and the mod had never asked him anything. `bc_knight_pulls_him_out` grows a branch on the `bc_flank_collapses` pattern, `else` arm unchanged. Two things are new: `bc_battle_open_effect` is **hoisted before the raise** so `scope:bc_chronicle_subject` carries the commander across the `trigger_event` boundary (`scope:combat_side` does not), and **every rung and ledger write in the event is wrapped in that scope** — a knight carries none of the army modifiers, and writing them bare would hang them where nothing reads them | 1 event, 4 options |
| knight, the two men who matter | **`has_relation_*` appeared in no gate anywhere in the mod** — the rivalry ladder *writes* relations and nothing has ever read one. Friend of his own commander (no side row), rival of his own commander (`ordered`/`down`, the cell that hands out nothing by design, and it must not read as cowardice — `bc_knight_shames_himself` owns that ground), and rival of the commander **opposite** while winning. The third is the first time a knight is allowed to notice who is in command on the other side; every shipped use of `enemy_side.side_commander` was rooted on a commander | 3 templates, base 80 |
| commander, what the men think of him | **`dread` appeared nowhere in the mod** — not a gate, not a `chance` row, not a trigger. It is the one number that is *about* the relation between a commander and his own ranks, and temperament does not stand in for it: a mild man who hanged a town has dread, a loud one who never did anything does not. `>= medium_dread` while losing (the line holds), `<= low_dread` while losing (it comes apart in pieces), `>= major_dread_value` with no side row at all (he does not have to ask). Thresholds are the vanilla **named** values, checked against AGOT's override of the same file — 20/40/70 both sides | 3 templates, base 40, 30, 30 |
| the same name on both sides | `is_close_family_of` appears sixteen times and **the house appeared nowhere** — the mod read kin and stopped there, which is the wrong place to stop in a setting whose wars are fought between branches far more often than between brothers. `bc_foe_is_house_trigger` is **disjoint from the kin gate by construction**, not a widening of it: its final `NOT` excludes close family, so the two cases stay two cases. House and not dynasty, because the cadet branches are the story and both comparisons have AGOT precedent. **Both houses are `exists`-guarded** — a lowborn has no house, CK3 does not short-circuit, and `bc_knight_no_house_behind_him` gates on exactly those men existing. Commander: he wins and declines to press, which is `ordered`/`down`, the cell that hands out nothing — the cost of the decision is the `bc_pursuit` he does not get. Knight: the same fact from the branch end, a man who cannot afford to be overlooked, with no side row because the audience is on the other side of the field either way | 1 trigger, 2 templates, base 40, 80 |
| where he learned it, and the man with no house | the education axis appeared **once in the whole mod, in a comment, and in no gate** — thirty-six commander templates read martial, prowess, temperament, dread, stress, station, age and the health traits, and none asked what the man was *raised* to do. Education is not martial: martial is how good he is now, the education trait is where he started. `education_martial_4` while winning (`ordered`/`up`, the top rung only — half a Westerosi host would clear `_3`), and the `NOR` of all four martial keys while losing (`measured`/`down`, which grants nothing on purpose: he is very often the cleverest man on the field, applying a good mind to a problem it was never shaped for). The keys are **listed, not grouped** — AGOT's override defines them at :480/:524/:561/:599 with `category = education` and no `group` line, so there is no group key to test. On the knight side `is_lowborn` had never been read either: the hedge knight on a winning field, **the largest prestige a knight template hands out (150)** because it is the only currency a man with no house has. Birth is not tenure, so this is not the station axis again | 3 templates, base 30, 30, 80 |
| the eve reads the man | nine eve options and **every contextual one gated on something outside him** — three on the ground, three on who is across the field. `j` asks what his own men think of him (`dread >= medium_dread`) and grants `bc_pursuit` alone, which only `f` had ever granted and only paired with `bc_spent_horse` behind a ground gate: same modifier, different reason, and `f` spends the horses while this spends him. `k` asks what the war has already cost him (`stress_level >= 2`) and is **the only option in the mod that buys relief instead of a posture** — no modifier at all, `add_stress = -15` for 50 prestige, twice what `c` pays for nothing so that it does not strictly dominate `c`. Neither touches a saved scope, so neither can go stale on a late-answered eve | 2 options, 9 → 11 |
| the lifesaver | twenty-four templates shipped since plan 1 and **only the nemesis memory left anything permanent** — and that one runs commander-to-commander. This is the first thing the mod writes between a knight and the man commanding him: a memory the *rescued* man carries, plus `bc_lifesaver_opinion` at `0.02` a month, the slowest decay in the file. The knight gets prestige and the `participants` tag; he does not get his own copy, because doing the pulling is a career and being pulled out is one day | 1 template, 1 type, 1 modifier |

**All six ground bands are now gated on as well as narrated.**
`bc_scripted_triggers.txt` wrapped four of six with a note to add the rest when
something needed them; `bc_the_broken_ground_holds` and `bc_the_green_closes_in`
are that something, and the wrapper set now matches `bc_ground_triggers.txt` one
for one.

**The two new foe triggers are not a ladder.** `bc_foe_is_countryman_trigger`
and `bc_foe_is_heathen_trigger` each test one property of the man opposite, so
unlike the arms of `bc_write_bond_effect` they are not mutually exclusive — a
foe can be both, and both templates can be valid on the same roll. The bond
*clause* still picks one, because that is a ladder; the *gates* are not.

**Every (KIND, SWING) pair the seam defines is now reached.** Plan 1 recorded
`entrenched`/`down`, `wild`/`up` and `ordered`/`down` as unreachable; later
content batches reached the first two, and this session's knight trait batch
reached the third. The pair census and the modifier reach counts in
`bc_ledger_effects.txt` are re-counted mechanically at the end of every batch —
plan 1's eighteen call sites are now **eighty-eight**, its "3, 3, 4 and 3" are
19, 14, 20 and 10, and the fall-through set is twenty-five sites rather than five. The
table's *behaviour* is unchanged; only its comments went wrong, and wrong in the
direction that invites a bad fix, which is why the counting command is now
written into the comment that carries the numbers.

**Template totals are now commander 39, knight 28.** Weight against vanilla's
`commander_none` at 1000: roughly 400 (the calibrated seven) + 300 + 180 + 70
+ 90 + 110 + 100 + 70 + 60; on the knight side, twenty-eight templates against
`knight_none`'s 2000 — twenty-seven at 80 and `bc_knight_pulls_him_out` at 60,
held below the rest because it is the one knight moment that writes something
permanent.
If a sweep shows the original seven crowded out, **the character batch is the
first to cut** — it is the newest and the least measured.

**Pooling is `random_valid`, and it is vanilla in three separate shapes.**
`00_animal_custom_loc.txt` for a customizable_localization; `birth.9002`
(`events/birth_events.txt:3515-3614`) for a `random_valid` nested inside a
`first_valid` with a bare fallback below it, which is the pattern the eve now
uses and which only works because an empty pool is *not valid* and `first_valid`
moves past it; `chariot_race_ongoing_jp.0400` for bare untriggered `desc` lines
inside a pool. The memory types were already built this way and simply had
one-line pools.

**`validate` was wrong about pools and was fixed in the same session.** Its
arm-shape check demanded an unguarded `text` arm from every
customizable_localization definition including `random_valid` ones — but under
pooling an unguarded arm is valid on *every* render, so it competes with the
real prose and takes its share, and for `bc_bond_clause`'s empty-string fallback
that means deleting a clause that was available. The rule mandated a defect. It
is now suspended for pooled definitions, and the obligation moves to the mod,
which writes its fallback arm as a bare `trigger_if` — true when its limit is
false, so it is valid on exactly the absent and unknown-value paths.

## Not yet verified in game

- The shieldbrother memory rendering, and that `[shieldbrother.GetFullName]`
  resolves.
- The 2026-08-14 localization fixes: that
  `[bc_named_commander.GetNicknameNoTooltip]` resolves inside the award toast
  (including in the liege branch, where the scope has moved), and that the 26
  new `nick_agot_*_desc` keys render on hover in the character sheet with no
  `Duplicate localization key` line in `error.log`.
- The nickname award **under its real gate**. The path itself is proven (see the
  2026-08-14 run) but no commander has yet banked three notable wins with
  `martial >= 14` and no nickname unaided. `"BC: nickname awarded"` is a permanent
  marker, so `probe 'BC:'` will report it the first time it happens.
- **The nemesis relation itself, on a character screen.** `BC: nemesis new` fired
  32 times and `BC: nemesis rung grudge` 26, so `bc_rivalry_set_relation_effect`
  runs and picks the right rung — but a marker only proves the effect was
  *reached*, not that the relation *landed*. Plan 4 targets
  `scope:bc_rivalry_foe`, saved from `var:bc_nemesis`, on the strength of an
  argument about shipped syntax: `set_relation_X = var:Y` has **zero** uses in
  vanilla or AGOT, while `has_relation_X = var:Y` ships. That argument is
  untested. Open a commander who took a `BC: nemesis new` and look for a grudge
  against the right man with the reason *"beat him on the field"*. If the marker
  fired and no relation exists, the target form is wrong and must be re-derived.
- The rivalry game rule rendering: **Battle Chronicles: Rivalry** with three
  readable options on the rules screen.
- **Whether a `combat_side` survives a day.** The dead-knight read is now deferred
  into `bc_chronicles.0093`. `BC: dk deferred` counting while
  `BC: dk deferred fired` is absent means the side object was freed and the read
  must be re-hung on the commander character. Both counting with
  `BC: dk found dead` still 0 means a day is not enough, or knights are not dying.
- **The wound, at last measurably.** `"BC: wound"` has never fired, and until
  2026-08-15 that was starvation — it had one call site on an arm that resolved
  once per run. It now also hangs off the major arm (~45/run).
  `BC: wound gates passed` counting with `BC: wound` 0 means the
  `var:bc_battle_size >= 3` read rejects; both 0 while `BC: moment major` counts
  means the five-row outer limit does.
- **That the SWING row rejects.** It was inert until the major arm started feeding
  it wins. Roughly a third of what now reaches `bc_wound_effect` is `swing up`,
  and nothing has yet proven the row turns those away.
- **The decisive rate at the new floor.** `>= 4` scored 1 in 44 battles. The
  binding constraint is call-site rarity, not the floor — only 10 of 233 moments
  ever request `decisive` and 7 request it too early. Both numbers should be
  re-measured together, never one alone.
- **The spike fragment.** `bc_banner_falls_ground`, invoked as
  `[bc_chronicle_subject.Custom('bc_banner_falls_ground')]` from
  `bc_banner_falls_toast_desc`. The next run must show a Banner Falls toast
  carrying a clause that names the ground, and `probe 'BC: ground'` must report
  a band for the same battle. **It is no longer the last clause on that toast**
  and looking for it there is how this check gets read as failed: the desc runs
  literal, ground, bond, scale, and the two shared clauses were wired onto all
  ten toasts by Plan B task 4. Read the markers and the toast together:

  | markers | toast | verdict |
  |---|---|---|
  | a band | ground clause present, matches band | shape proven — Plan B tasks 5-14 unblocked. Tasks 2, 3 and 4 shipped without it, under the plan's AMENDMENT: a clause APPENDED to a desc that keeps its literal fails to "no improvement", while the core sets tasks 5-14 write REPLACE the literal and fail to an empty toast. |
  | a band | no ground clause | **not a mechanism failure** — vanilla ships this exact shape. `court_poet_poetry_desc` (`localization/english/court_positions_2_l_english.yml:150`) is `"[court_poet_from_task.Custom('GeneratePoem_Outro_Rhyming')]"`, rendered by the `send_interface_toast` at `common/court_positions/tasks/00_court_poet_tasks.txt:94-98` against a scope saved earlier in that same file (`00_court_poet_tasks.txt:47`) — a saved scope, a bare `Custom`, in a toast desc. Eleven other vanilla toasts do the same. So a blank clause means the **anchor** or the **variable**, not the mechanism. **The retry that used to be needed here is not:** `bc_bond_clause` now renders on the same toast off the same anchor, and `bc_write_bond_effect` writes on every path, so a bond sentence present with the ground clause absent isolates the **variable**, and all three clauses absent at once isolates the **anchor**. **The two inferences do not rest on the same thing and only one of them is fragile.** The bond-present one rests on `bc_write_bond_effect` writing bare on the caller's current scope, which is the anchor only because `bc_battle_open_effect` is its sole caller — the paragraph named `NOTE ON SCOPE` in `bc_context_effects.txt` says that much, and the rest is that effect's own opening line, an unconditional `save_scope_as = bc_chronicle_subject`, which makes its current scope the anchor on every path. Give that effect a second caller on any other scope and characters other than the commander start carrying `bc_battle_bond`, so a rendered bond clause stops ruling out the failure the row below describes — the anchor resolving to somebody who is not the commander — and therefore stops vouching for the anchor. The all-three-absent one is unaffected by a second caller — a second caller adds a write elsewhere, it does not remove the one `bc_battle_open_effect` already performs on the commander — and fails only if that effect itself stops running on the anchor. Only if all three are absent, retry with `bc_commander`, saved on the same character fifteen lines above the toast (`bc_commander_phase_events.txt:65`; the toast is at `:80-85`). Do not abandon the design on this symptom. |
  | a band | ground clause present, wrong band | the arms read the variable off the wrong character — the anchor resolves but not to the commander who holds the scratch. |
  | `BC: ground missing` | no ground clause | not a spike failure — `combat.location` did not resolve at a phase event. Fix the read site in `bc_write_ground_effect` and run again. |
- **The widened eve.** `bc_chronicles.0011` now assembles three paragraphs and
  can offer eleven options — three unconditional and eight gated. Nothing static
  proves a `first_valid` picks the arm intended, or that an `option` `trigger`
  hides rather than greys.

  | markers | screen | verdict |
  |---|---|---|
  | `BC: ground <band>` (write-path marker, not the eve's own read) | a third paragraph naming that band | the write path and the eve's `scope:bc_eve_ground` read agreed — both work. A mismatch between the write-path band and the rendered paragraph is itself informative: the scope saved the province but a subsequent battle overwrote `BC: ground <band>` while the scope still points to the first one. |
  | `BC: ground` for **any named band** — rough, forest, jungle, winter, desert | third paragraph reads "The field is what it is" | `scope:bc_eve_ground` is saved but the band triggers do not match through it — check the scope's type, not the triggers. This is a real failure: all five have named arms and none of them should reach the fallback. Desert is the one band the DIAG run's Q5 terrain census never produced (0/137), so an absent `BC: ground desert` proves nothing either way. |
  | `BC: eve ground missing` | third paragraph reads "The field is what it is" | correct behaviour: the ground scope never saved, the fallback renders as designed. Not a trigger bug. The insert may be in the wrong branch of `bc_eve_offer_effect` if this counts while `BC: eve offered` counts equally. |
  | `BC: ground open` or unresolved province | third paragraph reads "The field is what it is" | correct: open is the only band that legitimately rides the fallback, and an unresolved province also reaches it. |
  | any | option d visible on open ground | the `trigger` is not gating; every option in the mod is then suspect |
  | `BC: eve option ground` counts | — | the option is reachable and taken. There is no "offered but not taken" marker. |
- **The `bc_ground_held` modifier on a character screen.** It is the fifth
  family-2 modifier and the first added since the cleanup rule was written. The
  run must show it appear when option d is taken and be **gone** by the next
  battle of the same war. If it is still there, `bc_cleanup_effect` did not run
  or did not reach the new line.
- **The sixteen new templates, one marker each.** Every one writes
  `BC: fired <key>`, so `probe 'BC: fired'` is a census of which gates are
  reachable. A template at **zero** is a gate that never opens, and the three
  worth suspecting first are `bc_dust_and_thirst` (the desert band scored 0/137
  and 0/360 in two censuses — a statement about where these campaigns were
  fought, not about the map), `bc_the_wine_at_noon` (drunkard is rare on a man
  trusted with a host) and `bc_the_practised_eye` (`strategist` **and** a
  15,000-man host at once). A template that **dominates** is the other failure:
  the six character-batch gates are the widest in the mod and the batch header
  names them as the first weights to cut.
- **That the new knight cases are seen at all.** The file header records
  `base = 40` behind narrow gates producing **zero** fires across a session
  against `knight_none` at 2000. All six shipped knight additions sit at 80. If
  the second and third batches are at zero while the shipped three fire, the
  base is the reason and not the gates.
- **That a pooled `random_valid` inside the eve's `first_valid` falls through
  when empty.** This is the one structural claim of the prose work, and vanilla
  `birth.9002` is the whole of the evidence. The test is cheap: fight on winter
  ground and read the third paragraph. **Winter or desert text = the nesting
  works.** "The field is what it is" on a battle whose markers say winter means
  the empty-pool fallthrough does *not* work the way `birth.9002` implies, and
  every pooled arm has to go back to a flat `triggered_desc` chain.
- **That the pools actually vary.** Three eves in one game should not print the
  same foe paragraph three times, and a knight with several battle memories
  should not have them all read identically. Uniform output with the structure
  in place means `random_valid` is being read as `first_valid`.
- **The style-branched nickname ladder.** `BC: nickname style UNMATCHED` is a
  permanent marker and **must stay at zero** — it counts a `var:bc_style_last`
  holding a value none of the six ladders claims, which is the seam's KIND
  vocabulary and the ladder having drifted apart.
- **The foe-relative award.** `probe 'BC: award'` reports which of the two
  currencies paid. The three markers `piety` / `prestige` / `no creed` partition
  the notable wins that reached the award; `nemesis` is a subset of the first
  two. A run where `BC: award no creed` dominates means most notable wins carry
  no BC moment — not that the award is broken. Separate the two causes by
  crossing it against `BC: bond nofoe`, which counts only the leaderless-enemy
  case. `BC: award piety` at zero while `BC: creed heathen` counts is the real
  failure: the read is landing on the wrong character.
