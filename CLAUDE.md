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
| `machineborn_arena_medal_v1` | Arena Medal currency (a number) — Grand Arena's economy, spent on Arena Pulls (guaranteed Vanguard-set gear, otherwise unobtainable) | Persistent |

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
   leveling below). Each champion also carries a `rarity` (reusing the
   same Common/Rare/Epic/Legendary scale as gear/charms — all 6 are
   currently Legendary) and a `faction` tag. Factions
   (`FACTIONS`/`FACTION_TIERS`) work like a team-composition mirror of
   gear sets: `factionBonusStatsFor(teamIds)` counts how many fielded
   champions share a faction and, at 2/3-member thresholds, applies a
   flat stat bonus to the *whole squad* (not just that faction's
   members) via `applyFactionBonusToUnit` — same generalized-tier
   pattern as `SET_TIERS`/`GEAR_SETS`, computed once per battle in
   `newBattle`/`resumeBattle` alongside gear/level/ascension. Current
   factions: Ironclad (Vara, Wren), Blightkin (Mire, Nyx), Stormcallers
   (Kestrel), Deathmark (Rook) — only Ironclad/Blightkin can reach
   bonus2 today since the other two have just one member each; that's
   expected to fill in as more champions are added, not a bug.
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
   2pc, a stat held back for 4pc. A 9th set, **Vanguard**
   (`ARENA_GEAR_SET_ID`, defined in `GEAR_SETS` but deliberately left
   OUT of `GEAR_SET_IDS`), exists only as Grand Arena's Arena Pull
   reward — see "Galactic War" below/Content framework's Grand Arena
   entry.
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
11. **Galactic War** (`galactic`, see Content framework below): the
    "big generic farm" node — 5v5 across `GALACTIC_WAVES`' 5 waves, no
    retreat between them, each wave after the first spawning with a
    stacking War Fervor Attack Up (`GALACTIC_WAR_FERVOR_STEP`). Unlocks
    at Campaign Ch.10 alongside Dungeon. Pays bulk Scrap+Gold+XP with
    no specialist material of its own — the deliberate exception to
    every other mode's single-material rule.
12. **Grand Arena** (`grandarena`, see Content framework below): the
    "smart AI" mode — 3v3 against one of four pre-built archetype
    squads (`GRAND_ARENA_ARCHETYPES`: Aggro/Control/Stall/Burst) that
    actually pick targets and fire Actives on cooldown via a
    `chooseAction` hook on each enemy unit (`enemyAct` dispatches to it
    when present; every other enemy in the game has none and keeps the
    original simple-random-basic-attack AI unchanged — see
    `focusLowestHpChooseAction` / `controlChooseAction` /
    `stallChooseAction`). Stage cycles through the 4 archetypes forever
    (`archetypeForStage`/`ENCOUNTERS[id].archetypeCycle`, resolved in
    `wavesForBattle`) — still an infinite ladder like the others, just
    with a rotating opponent identity instead of a rotating raw number.
    Both the stage-select label and the team-select title name the
    upcoming archetype so counter-picking a squad is actually possible
    before committing. Unlocks at Campaign Ch.10 alongside
    Dungeon/Galactic War. Pays Arena Medals only (`ENCOUNTER_ARENA_
    MEDAL_REWARD`) — no gear/charms drop from victory itself (reward
    purity); Medals are spent in the Armory's Arena Shop on
    `pullArenaGear`, a normal rolled gear instance (random slot/rarity-
    floor-Epic/main stat/substats) forced to the Vanguard set instead
    of an independent set roll — see the gear-sets bullet above.

## Content framework

Every battle mode owns exactly ONE material identity — the point is
that maxing one mode's specialty naturally pushes you to farm a
*different* mode next, not the same one forever. **Campaign** is the
odd one out: it has no farmable specialty of its own (a small
Scrap+Gold trickle only) because its job is being the account-wide
**gate** — `CONTENT_UNLOCKS` in `index.html` — that unlocks the other
modes as you clear its chapters. Nothing else is gated by anything;
once Campaign clears the threshold, that mode is open forever.
**Galactic War** is the one deliberate exception to "exactly ONE
material identity": it's the "big generic farm" node by design, so it
pays bulk Scrap+Gold+XP instead of owning a new specialist material —
see its row below and the implementation notes. **Grand Arena** keeps
one material identity (Arena Medals) but is the one mode whose reward
is spent on a gear *set* found nowhere else, rather than gating a
stat axis or an economy shared with other modes.

| Mode (`ENCOUNTERS` id) | Squad | Progression | Unlocks at | Rewards |
|---|---|---|---|---|
| **Campaign** (`campaign`) | 3v3 | 10 fixed hand-authored chapters (`CAMPAIGN_CHAPTERS`) — **not** the infinite ladder, no stat compounding, replaying an old chapter is always the same fight | Always open | Scrap + Gold only (bootstrap) |
| **Tower** (`tower`) | 5v5 | Infinite stage ladder | Campaign Ch.3 | Gear (guaranteed) + Gear Rework Catalysts (rare, rides gear drops) + Scrap |
| **Faction Wars** (`faction`) | 4v4 × 3 waves | Infinite stage ladder | Campaign Ch.5 | Charms (rolled chance) + Charm Dust (guaranteed) |
| **Main Boss** (`mainboss`) | 3v3 | Infinite stage ladder | Campaign Ch.8 | Gold + XP (champion leveling) + Shards (level cap) |
| **Dungeon** (`dungeon`) | 4v4 | Infinite stage ladder | Campaign Ch.10 | Ascension Cores only |
| **Galactic War** (`galactic`) | 5v5 × 5 waves | Infinite stage ladder, no retreat between waves within one attempt | Campaign Ch.10 (same threshold as Dungeon — a "you finished Campaign" bonus node, not a fifth step in the unlock sequence) | Bulk Scrap + Gold + XP only — no gear/charms/shards/catalysts/Ascension Cores |
| **Grand Arena** (`grandarena`) | 3v3 vs. one of 4 AI archetype squads | Infinite stage ladder, stage number cycles through the 4 archetypes (`(stage - 1) % 4`) forever | Campaign Ch.10 (alongside Dungeon/Galactic War) | Arena Medals only — spent on Arena Pulls (Vanguard-set gear), never dropped directly |

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

Galactic War earns its place the same way, but on a different axis:
turn economy under escalating pressure rather than debuff mitigation.
Every wave after the first spawns already carrying a stacking "War
Fervor" Attack Up (`GALACTIC_WAR_FERVOR_STEP` = 15% per wave, applied
in `spawnWave` when `ENCOUNTERS[id].stackingBuffPerWave` is set) — on
top of the same infinite-ladder stage scaling every other mode already
uses. There's no retreat between `GALACTIC_WAVES`' 5 waves (reusing the
same `currentWaves`/`waveIndex` wave-transition path Faction Wars'
3 waves already use — team HP/buffs carry over, only enemies respawn),
so a clean, fast, high-burst/AoE clear reaches the finale before Fervor
stacks up much, while a slow, grindy clear is fighting later waves at
a real Attack disadvantage on top of their own escalation.

Grand Arena earns its place on yet another axis: it's the only mode
where the *opponent's decision-making* is the test, not a stat curve or
a debuff/turn-economy mechanic. Every other enemy in the game (Campaign
through Galactic War) uses the same simple AI — `pickEnemyTarget` picks
a uniformly random living target (respecting Provoke) and always uses
`actor.basic`. Grand Arena's four archetype squads instead carry a
`chooseAction(actor, {foes, allies})` on every unit, and `enemyAct`
dispatches to it when present: Aggro/Burst always fire an Active the
instant it's off cooldown and finish off the lowest-HP% living foe;
Control fires its AoE lockdown Active whenever available and otherwise
targets whichever foe currently hits hardest; Stall spends a support
Active on its own lowest-HP% squadmate before ever attacking. Because
the archetype identity is tied to stage number (Aggro/Control/Stall/
Burst cycle in that fixed order), Burst — last in the cycle — is always
first met at a higher stage multiplier than Aggro; its base numbers are
tuned down from what its "big single hit" identity would otherwise
warrant specifically to offset that, rather than letting compounding
stage scaling stack with an already-hard-hitting kit into something
uncounterable on a fresh squad's very first turn.

Implementation notes:
- **Fixed vs. infinite vs. cycling**: `ENCOUNTERS[id].fixedChapters`
  (an array of chapter template arrays) marks Campaign as non-scaling —
  `spawnWave` passes stage `1` to `scaledEnemyTemplate` for such
  encounters instead of the real stage, so a chapter's authored numbers
  ARE its difficulty, forever. `getSelectedStage`/`setSelectedStage`
  both cap at `fixedChapters.length` so there's no "Chapter 11" once
  all 10 chapters are cleared. `ENCOUNTERS[id].archetypeCycle` is
  Grand Arena's equivalent for picking *which* opponent squad a stage
  fights (`archetypeForStage`), while still scaling stats and rewards
  by the real stage number like an infinite ladder — the two flags are
  independent, resolved together in `wavesForBattle`. Tower/Faction
  Wars/Main Boss/Dungeon/Galactic War have neither flag and no stage
  cap (`Infinity`), the plain infinite-ladder case.
- **Unlock UI**: a locked node's start button is `disabled` and its
  stage label reads "Locked — Campaign Ch. N" (`isContentUnlocked`,
  wired into `renderStageLabels`) rather than being hidden — the gate
  is visible, not mysterious. Clearing the exact Campaign chapter that
  unlocks a mode surfaces it on the victory screen ("Tower unlocked!").
  For Grand Arena specifically, both the stage label and the
  team-select title also name the upcoming archetype (" — vs Control"),
  since counter-picking only works if you know the opponent before
  locking in your squad.
- **Reward purity**: every `ENCOUNTER_*_REWARD` / `ENCOUNTER_DROPS` /
  `GEAR_REWORK_CATALYST_CHANCE` table only has entries for the mode(s)
  that actually own that material — e.g. `ENCOUNTER_GOLD_REWARD` has
  only `campaign`, `mainboss`, and `galactic` keys (Galactic War is the
  one intentional multi-mode overlap — see above — never a specialist
  material like gear/charms/shards/Ascension Cores/Arena Medals).
  `ENCOUNTER_ARENA_MEDAL_REWARD` has only `grandarena`, and there is no
  `ENCOUNTER_DROPS.grandarena` entry at all — Grand Arena's gear payoff
  (`pullArenaGear`) is a manual Armory spend, never a victory-screen
  roll, so the reward line itself stays Medals-only. `endBattle` guards
  every reward block with `if (reward > 0)` so a mode that doesn't
  grant something never shows a useless "+0 X" line.
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

The content framework's full seven modes are built: Campaign (the
gate, 10 fixed chapters), Tower/Faction Wars/Main Boss/Dungeon (each
single-material specialists unlocked by Campaign progress), Galactic
War (the generic bulk-farm node, unlocked alongside Dungeon), Grand
Arena (the "smart AI" counter-picking node, also unlocked alongside
Dungeon), Champion leveling, Ascension, and Gear Rework — see "Systems
that exist today" and "Content framework" above.

The user's full target roster — **Main Boss, Faction Wars, Tower,
Dungeon, Galactic Challenge/War, Grand Arena** — is now complete.

**New confirmed direction (in progress): a champion-acquisition end
cycle layered on top of the content framework.** The user's framing:
unlocking/building better champions should unlock higher tiers of
existing content, which pay better rewards, which build better
champions — another turn of the same "push here to unlock pushing
there" loop the whole game is built on, but at the champion-roster
level instead of a single mode's material. Confirmed with the user:
- **This is a deliberate, scoped exception to "No-FOMO... no gacha
  currency"** at the top of this doc — the user explicitly chose real
  RNG summons over a deterministic pick-your-champion unlock. Keep it
  scoped: no stamina/timers, no real-money purchase, no duplicate loss
  (a summon should never feel wasted) — the "no-FOMO" spirit still
  applies to everything *around* the RNG, just not to the pull itself.
- Rarity is a **power tier** (like gear rarity), not just a cost/cosmetic
  label — a Legendary champion should be meaningfully stronger than a
  Common one, mirroring how `GEAR_RARITY_MULT`/`GEAR_SUBSTAT_COUNT`
  raise gear's ceiling by rarity.
- Factions are champion tags with team-composition bonuses — **built**,
  see the Champions bullet above (`FACTIONS`/`FACTION_TIERS`/
  `factionBonusStatsFor`).

Still to design/build, in rough order:
1. **New lower-rarity champions** (Common/Rare/Epic) to seed the summon
   pool and populate Stormcallers/Deathmark past 1 member each. Kit
   complexity should scale with rarity the same way gear substat count
   does — e.g. Common/Rare could ship without a `passive`/`signature`
   (both already render safely when absent - `parsePassive`/`slotInfo`
   handle a missing string). **`active2` is NOT currently optional** -
   `skillButtonHtml`/`renderActions` assume every champion has one and
   will throw on `ability.name` if it's undefined - either give every
   new champion a real (even weak) `active2`, or guard that render path
   first. Passives/signatures are NOT data-driven from the `passive`/
   `signature` string fields (those are UI-only flavor text) - actual
   behavior is hand-wired per champion id inside `wireEventHooks()`, so
   a new champion's passive/signature (if any) needs its own bespoke
   block there, same as the existing 6.
2. **Summon currency + Armory action**: a new currency (name TBD, kept
   distinct from the existing `machineborn_shards_v1` — that one is
   already Main Boss's level-cap resource, a same-named "Shards" would
   collide) spent on a rarity-weighted champion pull, reusing the
   `DROP_RARITY_WEIGHTS`-style convention. Likely owned by Main Boss
   (already the account's "grow your champions" mode — Gold/XP/Shards)
   as an additional reward, similar to how Galactic War/Grand Arena
   each already made one documented exception to strict reward purity.
3. **Roster-tier gating**: the mechanism connecting "better champions"
   to "higher tiers, better rewards" is not yet designed. Leading idea:
   a `ROSTER_UNLOCKS`-style gate (mirroring `CONTENT_UNLOCKS`'s
   pattern) keyed off highest-rarity champion owned, blocking further
   stage progress on the existing infinite ladders past some stage
   until met, with a reward-multiplier bump alongside it - reusing
   `stageRewardMultiplier`'s slot rather than inventing a parallel one.
   Needs confirming with the user before building.

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
