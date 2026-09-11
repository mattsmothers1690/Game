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
  This is now a literal content framework, not just a narrative: every
  battle mode owns exactly ONE material identity, and Campaign is the
  account-wide gate that unlocks each mode as you push it. See "Content
  framework" below.

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
| `machineborn_scrap_v1` | Scrap currency (a number) — Tower's gear economy | Persistent |
| `machineborn_stage_progress_v1` | `{ encounterId: highestStageCleared }` — `campaign` key doubles as the account-wide content-unlock gate | Persistent |
| `machineborn_gold_v1` | Gold currency (a number) — Main Boss / champion-leveling only | Persistent |
| `machineborn_shards_v1` | Shard currency (a number) — Main Boss / level-cap only | Persistent |
| `machineborn_champion_progress_v1` | `{ championId: { level, xp, ascension } }` | Persistent |
| `machineborn_ascension_core_v1` | Ascension Core currency (a number) — Dungeon's champion-power economy | Persistent |
| `machineborn_levelcap_v1` | Account-wide level-cap tier (a number) | Persistent |
| `machineborn_set_catalyst_v1` | Set Catalyst currency (a number) — Tower/Rework: Set only | Persistent |
| `machineborn_stat_catalyst_v1` | Stat Catalyst currency (a number) — Tower/Rework: main stat value only | Persistent |
| `machineborn_substat_catalyst_v1` | Substat Catalyst currency (a number) — Tower/Rework: substats only | Persistent |
| `machineborn_charmdust_v1` | Charm Dust currency (a number) — Faction Wars' charm economy (Scrap's counterpart) | Persistent |

Every inventory is a flat array of independent rolled instances keyed
by a generated `instanceId`; equipping references an instance by ID.
Exclusivity (an instance can only be equipped on one champion/slot at
a time roster-wide) is enforced by `unclaimedGearInstances` /
`unclaimedCharmInstances`, which exclude instances claimed elsewhere.

## Systems that exist today

1. **Champions** (`CHAMPIONS`, 6 total): Vara (Guardian), Mire
   (Plaguebearer), Kestrel (Controller), Rook (Executioner), Nyx
   (Contagionist), Wren (Sentinel). Each has basic/active1/active2/
   passive/signature — fixed kits, no build choices. Base stats are
   modified by equipped gear and by champion level (see Champion
   leveling below).
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
6. **Scrap economy** (Tower's currency, gear-only): salvage an
   unequipped gear instance for Scrap (rarity-scaled payout,
   `GEAR_SALVAGE_VALUE`). Spend Scrap on **Upgrade** (levels 0–12,
   `gearLevelMultiplier` = `1 + level*0.08` applied to main stat + all
   substats, cost scales with rarity × level) and **Reforge** (flat
   rarity-scaled cost, rerolls one random substat to a new stat+value —
   no guarantee, same farming philosophy as a paid retry).
7. **Charm Dust economy** (Faction Wars' currency, charm-only, Scrap's
   counterpart — `machineborn_charmdust_v1`): salvage an unequipped
   charm instance for Charm Dust (`CHARM_SALVAGE_VALUE`). Spend it on
   **Reforge** (`charmReforgeCost`, rarity-scaled) — a charm has only
   one rolled dimension (its proc `chance`), so Reforge rerolls that
   within the same rarity's range. No Upgrade equivalent for charms
   yet (see "Next up").
8. **Gear Rework** (Tower-exclusive, on top of Upgrade/Reforge): three
   targeted rerolls that each touch exactly ONE of a gear piece's
   three independent rolls (main stat *value* only — the stat identity
   is slot-locked; whole set; whole substat block) while leaving the
   other two untouched — for "great roll, wrong set" or "right set,
   dead substats" pieces Reforge alone can't fix (Reforge only rerolls
   one substat at a time, and never touches set or main stat). Spends
   its own scarce **Catalyst** currency instead of Scrap — Set
   Catalyst / Stat Catalyst / Substat Catalyst
   (`machineborn_set_catalyst_v1` / `_stat_catalyst_v1` /
   `_substat_catalyst_v1`, `GEAR_REWORK_COST` = `{set:1, mainStat:1,
   substats:2}`). Catalysts are a rare bonus (`GEAR_REWORK_CATALYST_
   CHANCE`) riding on an actual Tower gear drop — gear-adjacent
   materials that only ever show up alongside gear, never as an
   independent roll. `reworkGearSet` / `reworkGearMainStat` /
   `reworkGearSubstats` live in the Armory's gear inventory list
   alongside Upgrade/Reforge/Salvage.
9. **Champion leveling** (Main Boss's currency — Gold/XP/Shards, see
   Content framework below): per-champion banked XP
   (`machineborn_champion_progress_v1`, `{ championId: { level, xp,
   ascension } }`). Leveling up is a manual, deterministic, paid action
   (`levelUpChampion`, in the Armory's Champions panel) — spends banked
   XP (`xpToNextLevel`) + Gold (`levelUpCost`), no RNG, matching the
   Scrap-Upgrade pattern rather than the rolled-instance one. Only hp/
   atk/def compound with level (`levelStatMultiplier`, +5%/level); spd/
   crit/acc/res stay fixed, same convention as enemy stage-scaling.
   The level cap (`effectiveLevelCap`, base 40) is account-wide and
   rises in Shard-gated +5 steps (`levelCapShardCost`).
10. **Ascension** (Dungeon's currency — Ascension Cores, see Content
    framework below): a second, much slower per-champion power axis
    stored in the same `championProgress` record as level/xp. Ranks
    0–5 (`ASCENSION_RANK_MAX`), each a flat +8% hp/atk/def
    (`ascensionMultiplier`, `ASCENSION_STAT_GROWTH`) stacking
    multiplicatively on top of the level multiplier
    (`applyAscensionToUnit` runs right after `applyLevelToUnit`) — same
    only-hp/atk/def convention as level and enemy stage-scaling, reused
    rather than inventing a parallel stat-growth shape. `ascendChampion`
    is manual/deterministic/paid (Ascension Cores only, no Gold/XP
    needed), rising cost per rank (`ascensionCost`). Lives in the same
    Champions panel row as Level Up.

## Content framework

Every battle mode owns exactly ONE material identity — the point is
that maxing one mode's specialty naturally pushes you to farm a
*different* mode next, not the same one forever. **Campaign** is the
odd one out: it has no farmable specialty of its own (a small
Scrap+Gold trickle only) because its job is being the account-wide
**gate** — `CONTENT_UNLOCKS` in `index.html` — that unlocks the other
four as you clear its chapters. Nothing else is gated by anything;
once Campaign clears the threshold, that mode is open forever.

| Mode (`ENCOUNTERS` id) | Squad | Progression | Unlocks at | Rewards |
|---|---|---|---|---|
| **Campaign** (`campaign`) | 3v3 | 10 fixed hand-authored chapters (`CAMPAIGN_CHAPTERS`) — **not** the infinite ladder, no stat compounding, replaying an old chapter is always the same fight | Always open | Scrap + Gold only (bootstrap) |
| **Tower** (`tower`) | 5v5 | Infinite stage ladder | Campaign Ch.3 | Gear (guaranteed) + Gear Rework Catalysts (rare, rides gear drops) + Scrap |
| **Faction Wars** (`faction`) | 4v4 × 3 waves | Infinite stage ladder | Campaign Ch.5 | Charms (rolled chance) + Charm Dust (guaranteed) |
| **Main Boss** (`mainboss`) | 3v3 | Infinite stage ladder | Campaign Ch.8 | Gold + XP (champion leveling) + Shards (level cap) |
| **Dungeon** (`dungeon`) | 4v4 | Infinite stage ladder | Campaign Ch.10 | Ascension Cores only |

Dungeon is deliberately different in *kind*, not just reward: every
enemy in `DUNGEON_WAVE_1` leads with a debuff instead of raw damage
(Stun / Poison / Defense+Attack Down / Stun+Speed Down) rather than
scaling up plain stat numbers like the other three. A team with
Resistance or self-cleanse (Vara's Bulwark Slam) clears it far more
reliably than a same-power team without that — the "strategy pushes
you further than raw stats" hook the user asked for when this mode was
designed. Keep that principle for any future mode: a new mode earns
its place by testing something *different* (turn economy, debuff
mitigation, burst vs. sustain, counter-picking), not by being Tower
with bigger numbers.

Implementation notes:
- **Fixed vs. infinite**: `ENCOUNTERS[id].fixedChapters` (an array of
  chapter template arrays) marks Campaign as non-scaling — `spawnWave`
  passes stage `1` to `scaledEnemyTemplate` for such encounters instead
  of the real stage, so a chapter's authored numbers ARE its
  difficulty, forever. `getSelectedStage`/`setSelectedStage` both cap
  at `fixedChapters.length` so there's no "Chapter 11" once all 10
  chapters are cleared — Tower/Faction Wars/Main Boss/Dungeon have no
  such cap (`Infinity`), matching their infinite-ladder nature.
- **Unlock UI**: a locked node's start button is `disabled` and its
  stage label reads "Locked — Campaign Ch. N" (`isContentUnlocked`,
  wired into `renderStageLabels`) rather than being hidden — the gate
  is visible, not mysterious. Clearing the exact Campaign chapter that
  unlocks a mode surfaces it on the victory screen ("Tower unlocked!").
- **Reward purity**: every `ENCOUNTER_*_REWARD` / `ENCOUNTER_DROPS` /
  `GEAR_REWORK_CATALYST_CHANCE` table only has entries for the mode(s)
  that actually own that material — e.g. `ENCOUNTER_GOLD_REWARD` has
  only `campaign` and `mainboss` keys. `endBattle` guards every reward
  block with `if (reward > 0)` so a mode that doesn't grant something
  never shows a useless "+0 X" line.
- Champion leveling itself (Gold/XP spend, level curve, Shard-gated
  cap) is unchanged from when it was Boss/Wave/Champion-Trial-agnostic
  — see the leveling bullet above; only Main Boss feeds it now.
- The Warlord-tier enemy in Main Boss, Dungeon's Warden, and Campaign's
  Chapter 8/10 capstones all set `isBoss: true`, reusing the generic
  devour-a-debuff-for-Fury mechanic (`hollowKingMechanic` — not
  actually Hollow-King-specific, it triggers off `actor.isBoss`) rather
  than a parallel one.
- Naming history: this framework replaced the original `boss`/`wave`/
  `champion` encounters — Tower *is* the old Boss content (same
  `BOSS_WAVE` squad), Faction Wars *is* the old Wave content
  (`WAVE_1/2/3`), Main Boss *is* the old Champion Trial
  (`CHAMPION_TRIAL`, variable name kept as-is). Renaming an
  `ENCOUNTERS` id resets that mode's `stageProgress` (a fresh key) —
  fine pre-launch, would need a migration if done post-launch.

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
  scratchpad test scripts covering resume-from-save, full-clears on
  each content mode, custom squad selection, charm procs, gear
  substats/rolls, gear sets (2pc/4pc), Campaign chapter/unlock gating,
  and the Scrap/Charm Dust/Catalyst economies — all should finish with
  an empty `ERRORS: []`. A test script's own *hardcoded IDs and numeric
  expectations* can go stale as new features change baseline behavior
  (e.g. a button-id or gear-count assertion written before a node was
  renamed or battle drops existed) — that's a stale test, not a
  regression, as long as the actual error list stays empty once fixed.

## Git workflow

- Branch `claude/machineborn-handoff-l1aegx`, single feature commits,
  always syntax-checked + regression-tested before commit.
- Commit messages explain *why*, not what (the diff shows what).

## Next up (not yet built)

The content framework's first five modes are built: Campaign (the
gate, 10 fixed chapters), Tower/Faction Wars/Main Boss/Dungeon (each
single-material specialists unlocked by Campaign progress), Champion
leveling, Ascension, and Gear Rework — see "Systems that exist today"
and "Content framework" above.

The user's full target roster is **Main Boss, Faction Wars, Tower,
Dungeon, Galactic Challenge/War, Grand Arena** — the first four exist;
**Galactic Challenge/War** and **Grand Arena** are the two still to
build, one at a time per the user's explicit preference (build, test,
ship one before starting the next):
- **Galactic Challenge/War**: proposed design (confirmed with the
  user, not yet built) — one continuous multi-wave siege, no retreat,
  enemies gain a stacking buff each wave (punishes slow/grindy clears,
  rewards burst/AoE). No new specialist material — pays a bulk
  Scrap+Gold+XP payout instead, the "big generic farm" node. Since
  there's no server/multiplayer, this can't be real PvP - it's a
  single-player siege gauntlet, not a war against anything.
- **Grand Arena**: proposed design (confirmed with the user, not yet
  built) — PvE vs. pre-built AI archetype squads (Aggro/Control/
  Stall/Burst) that actually use their full kit intelligently (unlike
  the simple AI other content uses), so it tests counter-picking, not
  just stat totals. Reward: a new Arena Medal currency for an
  Arena-exclusive gear/accessory pool not available anywhere else.

Note: "Rework node" turned out to mean gear substat/set/stat-value
rerolls, not a champion build-choice system — champions still have
zero build choices, so a champion-facing Rework is not on the table
until one exists (see the champion build-choice bullet below).

Longer-term, deferred until asked for:
- A **Charm Upgrade** to pair with the new Charm Reforge — gear has
  both Upgrade (level a piece's rolled values) and Reforge; charms
  only have Reforge + Salvage so far. Revisit if charm power creep
  becomes an issue.
- A champion build-choice system (talent variants, stat-path choices,
  etc.) — nothing like this exists yet; a champion-facing Rework mode
  would need it first.
