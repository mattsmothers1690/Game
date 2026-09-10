# Machineborn — Project State & Scope

Browser-based, mobile-first, offline-capable turn-based collector RPG.
Single self-contained `index.html` (no build step, no external
dependencies, no server). Served via GitHub Pages at
`https://mattsmothers1690.github.io/Game/index.html`. Branch:
`claude/machineborn-handoff-l1aegx`.

## Design philosophy (do not violate without the user asking)

- **Mechanic-driven, not power-driven.** Raid: Shadow Legends is the
  mechanical reference point, but the goal is deliberately *not* a
  reskin. Where a system diverges from Raid, that divergence is
  intentional — see Charms/gear-sets below.
- **No-FOMO, offline-first.** No stamina/energy timers, no gacha
  currency, no p2w gate. Progress is always available if you're
  willing to farm/push for it. Never add a system that requires
  waiting on a timer or spending real money to unblock.
- **Everything is a rolled instance, never a guarantee.** Charms roll
  a proc-chance within a rarity-tied range. Gear rolls a main stat +
  a rarity-scaled number of substats. Set membership on a gear piece
  is a *third*, fully independent random roll. Reforging a substat is
  a paid reroll, not a guaranteed upgrade. Higher rarity/stage raises
  the ceiling; it never raises the floor. Keep this pattern for any
  new acquisition system.
- **The long-term progression shape** (the user's own framing, keep
  this front-of-mind for future content/economy work):
  > Content A rewards gear to push B, B rewards champs/levels to push
  > C, C rewards charms to push D, D rewards gold, etc. — an
  > interlocking loop, not a linear tower. Scaling is infinite, tied
  > to player progress. When stuck on one node, there's always another
  > node worth farming to unblock it. Not "new extreme content behind
  > the newest p2w material" — a natural farm-and-push cycle.
  Gear/Charm content exists today (see below). Champion leveling
  (Gold/XP/shards) does not exist yet — see "Next up."

## Architecture

- **One file, one IIFE.** All game code lives in the `<script>` at the
  bottom of `index.html`, wrapped in `(function () { 'use strict'; ...
  })();`. Nothing is attached to `window` — a debug script targeting
  internal state (`team`, `emit`, etc.) from outside must patch a
  temporary copy of the file with an explicit `window.__debug = {...}`
  hook; never add such a hook to the committed file.
- **Turn-meter combat.** `takeTurn(actor, isExtra)` is the single
  entrypoint for both normal and extra turns. `runUntilPlayerInput()`
  drives the loop; `advanceToNextActor()` picks who's next by turn
  meter then speed.
- **Event bus** (`on(eventName, fn)` / `emit(eventName, payload)`) is
  the *one* dispatch mechanism reused for champion signatures,
  passives, Charms, and gear Set Bonuses. Adding a new reactive
  mechanic should almost always mean emitting/consuming an existing
  event, not inventing a parallel system. Known events: `debuffApplied`,
  `buffApplied`, `attackResolved`, `damageTaken`, `basicAttackUsed`,
  `death`, `turnStart`, `buffsExtended`, `shieldBroken`, `activeUsed`.
- **Deferred extra-turn pattern.** Granting an extra turn mid-event
  (e.g. from inside `performAttack`) is unsafe/reentrant. Effects that
  want to grant one instead set `extraTurnPending = owner`, which is
  read and cleared at the one safe point after the triggering action
  fully resolves, in `playerChooseTarget`.
- **Generalized tier abstraction for gear sets.** `SET_TIERS =
  [['bonus2',2],['bonus4',4],['bonus6',6]]`. Each tier independently is
  either `{stats:{...}}` (flat bonus) or `{trigger, condition, chance,
  effect}` (event-bus effect) — lets a set front-load an effect at 2pc
  and hold a stat for 4pc, or the reverse. `setBonusStatsFor` and
  `wireSetBonuses` both iterate `SET_TIERS` generically; never add a
  parallel `bonus2Stats`/`bonus2Effect`-style special case.

## Data model / persistence (all localStorage, browser-scoped)

| Key | Contents | Lifetime |
|---|---|---|
| `machineborn_save_v1` | Mid-battle resume snapshot (team/enemy state, wave index, **stage**, log) | Cleared on battle end |
| `machineborn_equipment_v1` | Per-champion equipped gear/charm instance IDs | Persistent |
| `machineborn_charms_v1` | Charm instance inventory | Persistent |
| `machineborn_gear_v1` | Gear instance inventory | Persistent |
| `machineborn_scrap_v1` | Scrap currency (a number) | Persistent |
| `machineborn_stage_progress_v1` | `{ encounterId: highestStageCleared }` | Persistent |

Every inventory is a flat array of independent rolled instances keyed
by a generated `instanceId`; equipping references an instance by ID.
Exclusivity (an instance can only be equipped on one champion/slot at
a time roster-wide) is enforced by `unclaimedGearInstances` /
`unclaimedCharmInstances`, which exclude instances claimed elsewhere.

## Systems that exist today

1. **Champions** (`CHAMPIONS`, 6 total): Vara (Guardian), Mire
   (Plaguebearer), Kestrel (Controller), Rook (Executioner), Nyx
   (Contagionist), Wren (Sentinel). Each has basic/active1/active2/
   passive/signature. No leveling — stats are fixed base values,
   modified only by equipped gear.
2. **Buffs/debuffs**: full set including +/- Atk/Def/Spd/Res/Acc,
   Shield, Block Buffs, Block Debuffs, Stun, Poison, Bleed, Burn,
   Counter, Provoke, etc.
3. **Charms** (accessories, `CHARM_TYPES` + `CHARM_SLOT_COUNT = 3`):
   deliberately *not* stat sticks — each is one trigger+effect pair
   that fires on a rolled % chance (`CHARM_RARITY_RANGE`). This is the
   explicit break from a Raid-blessing copy the user asked for.
4. **Gear** (6 slots — weapon/helmet/shield/gauntlets/chestplate/
   boots, matching Raid's layout): each farmed piece is a rolled
   instance — fixed main stat per slot (`GEAR_MAIN_STAT`), a
   rarity-scaled roll (`STAT_ROLL_RANGE` × `GEAR_RARITY_MULT`), and
   1/2/3/4 substats by rarity (`GEAR_SUBSTAT_COUNT`).
5. **Gear sets** (`GEAR_SET_IDS`, 8 total: venomous, bulwark, tempo,
   executioner, shockwave, blight, zealot, tactician): which set a
   piece belongs to is a third independent roll at farm time. 4 are
   stat-led at 2pc with an event effect at 4pc (venomous also has a
   6pc capstone); the other 4 flip the pattern — a small-%-chance
   mechanical effect (Stun/Poison/extra-turn/cooldown-reduction) at
   2pc, a stat held back for 4pc.
6. **Scrap economy**: salvage an unequipped gear/charm instance for
   Scrap (rarity-scaled payout, `GEAR_SALVAGE_VALUE` /
   `CHARM_SALVAGE_VALUE`). Spend Scrap on gear-only **Upgrade** (levels
   0–12, `gearLevelMultiplier` = `1 + level*0.08` applied to main stat
   + all substats, cost scales with rarity × level) and **Reforge**
   (flat rarity-scaled cost, rerolls one random substat to a new
   stat+value — no guarantee, same farming philosophy as a paid
   retry).
7. **Battle rewards**: victory grants Scrap + independent gear/charm
   drop chances (`ENCOUNTER_DROPS`), both rolled from the same
   weighted-rarity table drops and manual farming share
   (`DROP_RARITY_WEIGHTS`).
8. **Infinite stage progression**: Boss and Wave are each an unbounded
   stage ladder, not a fixed fight. Enemy hp/atk/def compound
   `1.12^(stage-1)` (spd/crit/acc/res untouched, to keep turn economy
   and hit/crit math sane). Scrap reward scales `1 + (stage-1)*0.25`.
   Drop-rarity weights shift toward Epic/Legendary as stage rises
   (capped bump). Clearing a stage unlocks the next and permanently
   stays farmable — the "push then come back to farm" loop in
   miniature, on the two nodes that exist today. Stage is threaded
   through save/resume so a reload mid-fight restores the exact same
   difficulty instance.

Manual "Farm Gear" / "Farm Charms" buttons in the Armory remain as a
testing/manual shortcut alongside real battle drops — not the only
acquisition path anymore.

## Testing methodology

- **No test framework is committed to the repo.** Verification is
  done with disposable Playwright (`playwright-core`) scripts written
  to the session scratchpad (never committed), driving a real
  Chromium instance
  (`/opt/pw-browsers/chromium-1194/chrome-linux/chrome`) against the
  actual `file://` HTML.
- **Syntax-check first**, cheaply, before spinning up a browser:
  ```js
  const fs = require('fs');
  const html = fs.readFileSync('index.html', 'utf8');
  const match = html.match(/<script>([\s\S]*)<\/script>/);
  new Function(match[1]); // throws on a syntax error
  ```
- **For RNG-dependent effects** (a set-bonus proc chance, a charm
  proc chance, a drop rarity roll), do not conclude "broken" from one
  failed observation. Reason about the compound probability (trigger
  frequency × proc chance) before calling it a bug. If a single trial
  is inconclusive, either run more trials, force the specific
  triggering action (e.g. force a champion to use a specific Active
  instead of relying on auto-basic loops), or — if truly needed —
  patch a **temporary scratchpad copy** of the file with a
  `window.__debug` hook exposing `emit`/internal state directly, to
  isolate the wiring from RNG variance. Never leave such a hook in the
  committed file.
- **Regression habit**: after any change, re-run the standing set of
  scratchpad test scripts covering resume-from-save, boss/wave
  full-clear, custom squad selection, charm procs, gear substats/
  rolls, gear sets (2pc/4pc), and the Scrap economy — all should
  finish with an empty `ERRORS: []`. A test script's own *hardcoded
  numeric expectations* can go stale as new features change baseline
  behavior (e.g. a gear-count assertion written before battle drops
  existed) — that's a stale test, not a regression, as long as the
  actual error list stays empty.

## Git workflow

- Branch `claude/machineborn-handoff-l1aegx`, single feature commits,
  always syntax-checked + regression-tested before commit.
- Commit messages explain *why*, not what (the diff shows what).

## Next up (not yet built)

**Champion leveling** is the natural next system — it's the hinge in
the user's loop design that nothing currently feeds into. Needed
pieces:
- A new **Gold** currency (distinct from Scrap — Scrap is gear-only).
- XP/Level on each champion, with a stat-growth curve per level.
- A level cap gated by **Shards** (from a not-yet-built Champion
  content node) — this is what forces the "push another node to
  unblock this one" loop rather than a single linear grind.
- Gold's own source can reuse the stage-ladder pattern already built
  (a third `ENCOUNTER`-style node, or a per-stage bonus payout
  alongside existing Scrap).

Longer-term, deferred until asked for:
- A **Champion node** and **Rework node** (materials to reroll a
  champion's build choice, once such a choice exists beyond gear).
- Possibly renaming/re-theming Boss/Wave into the "Gear Trial" /
  "Charm Vault" node identity discussed in design chat — not done yet,
  current code still calls them `boss`/`wave`.
- Upgrading/reforging/salvaging exists for gear; charms have salvage
  only (no upgrade/reforge) — revisit if charm power creep becomes an
  issue.
