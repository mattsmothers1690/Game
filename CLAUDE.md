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
| `machineborn_summon_shard_v1` | Summon Shard currency (a number) — Main Boss's second reward, spent on champion Summon pulls | Persistent |
| `machineborn_champion_roster_v1` | Array of owned champion ids — only `STARTER_CHAMPION_IDS` start owned; everything since must be summoned | Persistent |

Every inventory is a flat array of independent rolled instances keyed
by a generated `instanceId`; equipping references an instance by ID.
Exclusivity (an instance can only be equipped on one champion/slot at
a time roster-wide) is enforced by `unclaimedGearInstances` /
`unclaimedCharmInstances`, which exclude instances claimed elsewhere.

## Systems that exist today

1. **Champions** (`CHAMPIONS`, 14 total): the original six — Vara
   (Guardian), Mire (Plaguebearer), Kestrel (Controller), Rook
   (Executioner), Nyx (Contagionist), Wren (Sentinel), all Legendary
   rarity with the full basic/active1/active2/passive/signature kit —
   plus eight added since to seed the summon pool: Common/Rare pairs
   Zephyr/Squall (Stormcallers) and Fang/Vex (Deathmark), and one Epic
   per faction — Bastian (Ironclad), Morwen (Blightkin), Talon
   (Stormcallers), Raze (Deathmark). No build choices on any of them.
   Base stats are modified by equipped gear and by champion level (see
   Champion leveling below).
   - **Starters are deliberately weak, not the original six** — see
     "New-game onboarding" below. `STARTER_CHAMPION_IDS = ['zephyr',
     'fang', 'squall']` (2 Common + 1 Rare). The original six Legendaries
     now live entirely in the real Summon pool alongside Vex/Bastian/
     Morwen/Talon/Raze; Bastian and Vara are additionally handed out as
     guaranteed, deterministic Campaign Chapter 1 stage-clear rewards
     (`CAMPAIGN_STAGE_CHAMPION_REWARDS`) rather than only via RNG.
   - **Rarity** reuses the same Common/Rare/Epic/Legendary scale as
     gear/charms rather than a parallel one. **Kit complexity scales
     with rarity**, mirroring how `GEAR_SUBSTAT_COUNT` scales gear's
     substat count: Common ships with basic+active1 only, Rare adds
     active2, Epic adds a passive on top of that, Legendary (the
     original six) has the full basic/active1/active2/passive/
     signature kit — the rarity ladder is now fully populated at every
     tier. A missing `active2`/`passive`/`signature` renders safely
     (the button/info-tab for it is simply omitted) — `renderActions`,
     `slotInfo`, and `parsePassive` all check for presence first; this
     was NOT true before Zephyr/Fang shipped (`skillButtonHtml('active2',
     actor.active2, ...)` used to assume every champion had one and
     would throw otherwise) — a real fix, not speculative. Passives/
     signatures are still NOT data-driven from the `passive`/`signature`
     string fields (those are UI-only flavor text) — real behavior is
     hand-wired per champion id inside `wireEventHooks()`. All four Epic
     champions have a real passive block there (Bastian: Shield on a
     hurt ally via `damageTaken`; Morwen: Attack Up on a Poisoned kill
     via `death`; Talon: Speed Up to a random ally via `activeUsed`;
     Raze: an extra turn on finishing a debuffed enemy via
     `attackResolved`, reusing the same `extraTurnPending` mechanic
     Zealot's gear-set 2pc already uses) — each a single, simple hook,
     appropriately smaller in scope than a Legendary's passive+signature
     pair. Zephyr/Squall/Fang/Vex still have none.
   - **Factions** (`FACTIONS`/`FACTION_TIERS`) work like a
     team-composition mirror of gear sets: `factionBonusStatsFor(teamIds)`
     counts how many fielded champions share a faction and, at 2/3-member
     thresholds, applies a flat stat bonus to the *whole squad* (not
     just that faction's members) via `applyFactionBonusToUnit` — same
     generalized-tier pattern as `SET_TIERS`/`GEAR_SETS`, computed once
     per battle in `newBattle`/`resumeBattle` alongside gear/level/
     ascension. Every faction now has exactly 3 members — Ironclad
     (Vara, Wren, Bastian), Blightkin (Mire, Nyx, Morwen), Stormcallers
     (Kestrel, Squall, Zephyr, Talon — 4), Deathmark (Rook, Vex, Fang,
     Raze — 4) — so bonus2 *and* bonus3 (verified: fielding 3 Deathmark
     champions together raises Rook's critDamage by the bonus3 amount
     on top of bonus2, since every squad size in the game is 3+) are
     both reachable in every faction, no longer a dormant tier.
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
13. **Champion summoning + Roster Tier** (the champion-acquisition end
    cycle the user asked for, layered on top of everything above): Main
    Boss's second reward, Summon Shards, buys a rarity-weighted
    champion pull in the Armory's Summon panel (`pullChampionSummon`) —
    see the Champions bullet and the data-model table for the currency/
    ownership keys. `rosterTier()` (1/2/3, thresholds at 6/8/10 owned
    champions) feeds `rosterStageCap()` (15/30/∞), which
    `maxSelectableStage` applies to every non-`fixedChapters` encounter
    (Tower/Faction Wars/Main Boss/Dungeon/Galactic War/Grand Arena) —
    Campaign is exempt (its own `fixedChapters` cap already governs it,
    and it's the account-wide gate, not something roster growth should
    touch). Deliberately keyed off **roster size**, not highest rarity
    owned, despite that being the originally-approved idea: the starter
    six are already Legendary, so a rarity-keyed gate would sit at max
    tier from turn one and never actually gate anything — roster size
    grows exactly when a summon lands a genuinely new champion, which
    is the behavior this is meant to reward. The cap only holds back
    the stage *stepper* (`getSelectedStage`/`setSelectedStage`, both now
    thin wrappers around `maxSelectableStage`) — it never erases
    `stageProgress` already banked, and no separate reward-multiplier
    was added alongside it: `stageRewardMultiplier` already scales hard
    with the stage number a higher tier unlocks access to, so "better
    champions unlock higher tiers which pay better rewards" falls out
    of mechanics that already existed. A capped stage shows "(roster-
    capped)" in its label (`renderStageLabels`) rather than the plain
    "(new)" tag, and the Armory's Summon panel shows the current tier/
    cap directly (`rosterTierDisplay`) — same "the gate is visible, not
    mysterious" convention as `isContentUnlocked`.
14. **New Game / reset** (`resetGame`, `allSaveKeys`): battle-in-
    progress save (`machineborn_save_v1`) already existed; this is the
    account-progress counterpart — every persistent key the game
    writes (all 18: equipment/gear/charm inventories, every currency,
    champion progress/roster, level cap, stage progress, the mid-battle
    save) in one list, referenced by each key's own `_KEY` constant
    rather than retyped as a string literal so a future new economy
    can't silently be left out of a reset. `allSaveKeys` is a function
    (not a top-level array) purely so it can be declared anywhere in
    the file regardless of where each `_KEY` constant is assigned —
    it only reads their values when actually called, well after the
    whole script has initialized. A "NEW GAME (reset all progress)"
    button on the start screen (deliberately small/muted/separated
    from the node buttons, `#newGameBtn`) confirms via the native
    `confirm()` dialog before calling `resetGame`, which removes every
    key and reloads — reload is what actually re-triggers every
    module-level `loadXxx()` fallback-to-default (`loadChampionRoster()`
    → `STARTER_CHAMPION_IDS`, `loadScrap()` → `0`, etc.), the same
    fresh-save behavior already exercised by every test in this repo's
    testing methodology (`localStorage.clear()` + reload). No new
    persistence mechanism was needed - resetting is just "remove
    everything, let the existing defaults do their job."

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

| Mode (`ENCOUNTERS` id) | Squad | Progression | Unlocks at (Campaign stage) | Rewards |
|---|---|---|---|---|
| **Campaign** (`campaign`) | 3v3 | 22 fixed hand-authored stages (`CAMPAIGN_CHAPTERS` = `CAMPAIGN_CH1_STAGES` (12) + the old `CAMPAIGN_CH1..CH10` (10), now displaying as Chapter 2-11 — see "New-game onboarding" below) — **not** the infinite ladder, no stat compounding, replaying an old stage is always the same fight | Always open | Scrap + Gold only (bootstrap) |
| **Tower** (`tower`) | 5v5 | Infinite stage ladder | Stage 7 | Gear (guaranteed) + Gear Rework Catalysts (rare, rides gear drops) + Scrap |
| **Faction Wars** (`faction`) | 4v4 × 3 waves | Infinite stage ladder | Stage 10 | Charms (rolled chance) + Charm Dust (guaranteed) |
| **Main Boss** (`mainboss`) | 3v3 | Infinite stage ladder | Stage 4 | Gold + XP (champion leveling) + Shards (level cap) + Summon Shards |
| **Dungeon** (`dungeon`) | 4v4 | Infinite stage ladder | Stage 22 (Campaign complete) | Ascension Cores only |
| **Galactic War** (`galactic`) | 5v5 × 5 waves | Infinite stage ladder, no retreat between waves within one attempt | Stage 22 (same threshold as Dungeon — a "you finished Campaign" bonus node, not a fifth step in the unlock sequence) | Bulk Scrap + Gold + XP only — no gear/charms/shards/catalysts/Ascension Cores |
| **Grand Arena** (`grandarena`) | 3v3 vs. one of 4 AI archetype squads | Infinite stage ladder, stage number cycles through the 4 archetypes (`(stage - 1) % 4`) forever | Stage 22 (alongside Dungeon/Galactic War) | Arena Medals only — spent on Arena Pulls (Vanguard-set gear), never dropped directly |

`CONTENT_UNLOCKS = { mainboss: 4, tower: 7, faction: 10, dungeon:
CAMPAIGN_CHAPTERS.length, galactic: CAMPAIGN_CHAPTERS.length,
grandarena: CAMPAIGN_CHAPTERS.length }` — keyed to Campaign's flat
stage index now, not a "chapter number" (Chapter 1 alone spans 12
stages). Main Boss deliberately unlocks *before* Tower despite Tower's
old head-start, because Main Boss's 3v3 matches the 3-champion starter
roster exactly while Tower's 5v5 doesn't — see "New-game onboarding".

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

**New-game onboarding — Campaign Chapter 1 as a 12-stage tutorial
ladder.** The user's framing: a new save should open with a weak
starter squad, learn the game's mechanics across Campaign's first
chapter, earn two guaranteed champions along the way, then hit a real
plateau that forces a detour into whichever other mode just unlocked —
the core farm-and-push loop starting from the very first session
instead of only after Campaign is fully cleared.
- `CAMPAIGN_CH1_STAGES` (`CAMPAIGN_C1S1`..`CAMPAIGN_C1S12`) are 12 flat
  `fixedChapters` entries prepended to the old `CAMPAIGN_CHAPTERS`
  array (`CAMPAIGN_CHAPTERS = CAMPAIGN_CH1_STAGES.concat([CAMPAIGN_CH1,
  ..., CAMPAIGN_CH10])`, 22 stages total) — this needed zero changes to
  the core stage-indexing machinery (`getSelectedStage`/
  `setSelectedStage`/`maxSelectableStage`/`spawnWave` all already
  operate on a flat stage index). A new per-encounter `stageLabelFor`
  hook (`campaignStageLabel`, mirroring Grand Arena's existing
  `archetypeCycle` hook pattern) translates the flat index back into
  "Chapter 1 - Stage N/12" for stages 1-12 and "Chapter M" (M =
  stage-12+1, so old Chapter 1 now reads "Chapter 2") beyond that —
  `campaignClearMessage` produces the matching victory-screen text
  ("Stage N cleared", "Chapter 1 cleared — Chapter 2 unlocked!", etc.).
- Stages 1-3 are trivial (teach basic attack/targeting/that debuffs
  exist). Stage 4 unlocks Main Boss — the earliest detour, sized to
  match the 3-champion starter roster exactly. Stage 6 grants Bastian
  (Epic) and Stage 8 grants Vara (Legendary) directly via
  `unlockChampion()` — see `CAMPAIGN_STAGE_CHAMPION_REWARDS = {6:
  'bastian', 8: 'vara'}` — **not** an RNG Summon pull. This was a
  deliberate confirmed decision: the user first asked whether the 1st/
  2nd Summon pulls could simply bypass RNG and auto-grant a champion;
  since Summon Shards aren't even earnable yet this early (they're a
  Main Boss reward, and Main Boss only just unlocked at Stage 4), a
  direct stage-clear reward was simpler and just as effective, so that
  path was dropped in favor of a plain `unlockChampion()` call — no new
  "Champion Core" item, no RNG-bypass machinery. Stage 7 unlocks Tower
  (5v5) — timed so the roster has reached exactly 5 owned champions
  (3 starters + Bastian + Vara) by the time a 5v5 mode needs fielding.
  Stage 9 is the intentional plateau (see below). Stage 10 unlocks
  Faction Wars. Stage 12 is Chapter 1's capstone fight.
- **Difficulty tuning was empirical, not assumed** — same methodology
  as the historical Chapter 8/10 multi-pass tuning. Stage 9 was
  deliberately built as a wall the starter+Bastian+Vara squad cannot
  reliably clear (verified via repeated-trial probes, not a single
  observation — a first pass came back an easy 12/12 because the
  squad already has 5 champions by Stage 9, not the original 3, so the
  wall had to be raised until real trials showed it holding). Stages
  10-12 were then re-tuned around that same probe methodology so the
  curve reads as "hard wall → moderate relief → moderate wall →
  capstone" rather than two back-to-back bricks. Exact numbers live in
  the `CAMPAIGN_C1S*` blocks' own comments — treat them as a first
  pass, expect further iteration once more of the loop (gear from
  Tower, XP from Main Boss) is actually available to playtest against.
- **Main Boss's own base difficulty (`CHAMPION_TRIAL`) was retuned
  down** for the same reason — it used to be reachable only very late
  (the old Chapter 8 gate), tuned around a leveled/geared 6-Legendary
  squad, but now unlocks at Stage 4 when the roster is still exactly
  the 3 level-1 starters with no gear. Verified via probe at ~1/8 wins
  before the retune, ~8/8 after (Main Boss stage 1 is meant to be the
  easy "farm to power up" escape valve, not a wall — the wall is
  Chapter 1 Stage 9's job).
- `openTeamSelect` now filters `ENCOUNTERS[id].teamIds` through
  `isChampionOwned` before pre-selecting a default squad — needed once
  starters stopped being "whichever 3-6 champions a mode's `teamIds`
  names," since an unowned default member would otherwise silently
  count toward a "full" squad.
- Chapters 2-11 (the old Chapters 1-10) keep their original structure
  (one hand-authored fight per chapter, no multi-stage sub-ladder) but
  their **numbers were retuned** — see "Chapters 2-11 difficulty seam"
  below. Whether they also get the multi-stage tutorial-ladder
  treatment Chapter 1 got, or stay single capstone fights between the
  other modes' ladders, is still an explicitly open decision the user
  has not yet made (see "Next up").

**Chapters 2-11 difficulty seam — retuned.** The original CH1-CH10
numbers were hand-tuned for a level-1, all-Legendary 3-champion squad
(~200 total atk). Once Chapter 1 flipped the starters to a weak
Common/Rare trio, the squad actually arriving at Chapter 2 is whichever
3 of the 5 post-Chapter-1 champions (3 starters + Bastian + Vara) the
player picks — best case Vara+Bastian+Squall, ~163 total atk,
meaningfully less burst than the original baseline despite similar
HP/DEF. Probed (`probe_ch2_11.js`, 8 trials/stage, that exact squad, no
extra gear/levels — the same pessimistic-floor methodology Chapter 1
used) before touching anything: the original numbers came back 8/8 for
Chapters 2-8 (no climb at all) then fell off a cliff to 3/8 at Chapter
9's boss and 1/8 at both Chapter 10 (a plain chapter, not even a
capstone) and Chapter 11's finale — backwards pacing where a non-boss
chapter was harder than the boss before it.
- First retune pass overshot hard in the other direction — an
  aggressive ~8-15%/chapter exponential climb hit a wall as early as
  Chapter 5 (0/8) and stayed at 0/8 through Chapter 11, because this
  engine's naive-auto-battle combat has a narrow, fairly binary
  win/loss threshold in raw stat space (small stat bumps can flip a
  matchup from ~100% to ~0%), not a gradual slope — a lesson worth
  keeping in mind for any future difficulty tuning here: move in small
  increments and re-probe, don't extrapolate a whole curve from theory.
- Landed on (verified via repeated probes, accepting normal 8-trial
  sample noise): Chapters 2-6 easy/warm-up (~8/8), Chapter 7 a real dip
  (~4-7/8, texture rather than a flat floor), Chapter 8 building up,
  Chapter 9's boss a genuine wall (~0-1/8, comparable to Chapter 1's
  own Stage 9), Chapter 10 a real relief step (~5-8/8, still noticeably
  harder than 2-6 thanks to a Bleed-heavy enemy kit that punishes lack
  of cleanse/Resistance even at lower raw stats than the finale), and
  Chapter 11's finale the hardest overall (~2-3/8) — the same
  wall/relief/wall/capstone shape Chapter 1's own plateau established,
  now recurring one tier up. `test_content_framework.js`/
  `test_galactic_war.js` (which clear the Chapter 11 finale with a
  stronger Vara+Mire+Kestrel squad to test the dual Dungeon/Galactic
  War unlock) still pass — the retune only pulled numbers down/up
  within a narrow band, it didn't change the finale's role.
- Exact numbers live in the `CAMPAIGN_CH1`-`CAMPAIGN_CH10` blocks'
  own comments (note: these old variable names now display as Chapter
  2-11 — see the naming-history bullet under Implementation notes).
  Treat this as a second pass, not final — same as Chapter 1's own
  numbers, expect further iteration once Tower gear and Main Boss
  levels are actually in the loop when a real player reaches this
  point instead of the zero-farming floor this probe assumes.

Implementation notes:
- **Fixed vs. infinite vs. cycling**: `ENCOUNTERS[id].fixedChapters`
  (an array of chapter template arrays) marks Campaign as non-scaling —
  `spawnWave` passes stage `1` to `scaledEnemyTemplate` for such
  encounters instead of the real stage, so a chapter's authored numbers
  ARE its difficulty, forever. `getSelectedStage`/`setSelectedStage`
  both cap at `fixedChapters.length` (now 22) so there's no "Stage 23"
  once every stage is cleared. `ENCOUNTERS[id].archetypeCycle` is
  Grand Arena's equivalent for picking *which* opponent squad a stage
  fights (`archetypeForStage`), while still scaling stats and rewards
  by the real stage number like an infinite ladder — the two flags are
  independent, resolved together in `wavesForBattle`. Tower/Faction
  Wars/Main Boss/Dungeon/Galactic War have neither flag and no stage
  cap (`Infinity`), the plain infinite-ladder case.
- **Unlock UI**: a locked node's start button is `disabled` and its
  stage label reads "Locked — Campaign " + `campaignStageLabel(...)`
  (e.g. "Locked — Campaign Chapter 1 - Stage 7/12", `isContentUnlocked`,
  wired into `renderStageLabels`) rather than being hidden — the gate
  is visible, not mysterious. Clearing the exact Campaign stage that
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
  roll, so the reward line itself stays Medals-only.
  `ENCOUNTER_SUMMON_SHARD_REWARD` has only `mainboss` — Main Boss's
  champion-growth identity now covers Gold/XP/Shards *and* Summon
  Shards, but `pullChampionSummon` is the same kind of manual Armory
  spend as Grand Arena's gear pull, never a victory-screen roll.
  `endBattle` guards every reward block with `if (reward > 0)` so a
  mode that doesn't grant something never shows a useless "+0 X" line.
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

**Champion-acquisition end cycle layered on top of the content
framework — built.** The user's framing: unlocking/building better
champions should unlock higher tiers of existing content, which pay
better rewards, which build better champions — another turn of the
same "push here to unlock pushing there" loop the whole game is built
on, but at the champion-roster level instead of a single mode's
material. All three confirmed pieces are done — see Systems item 13
and the Champions bullet above for the full mechanism:
- Real RNG champion summons (`pullChampionSummon`), a deliberate,
  scoped exception to "No-FOMO... no gacha currency" at the top of this
  doc — the user explicitly chose this over a deterministic pick-your-
  champion unlock. Kept scoped: no stamina/timers, no real-money
  purchase, no wasted pull (a duplicate refunds instead of doing
  nothing) — the no-FOMO spirit still applies to everything *around*
  the RNG, just not to the pull itself.
- Rarity as a **power tier** (like gear rarity) via kit-complexity-
  scales-with-rarity (Common/Rare/Epic/Legendary), not just a cost/
  cosmetic label.
- Factions as champion tags with team-composition bonuses
  (`FACTIONS`/`FACTION_TIERS`/`factionBonusStatsFor`).
- Roster Tier (`rosterTier`/`rosterStageCap`/`maxSelectableStage`)
  connecting "better champions" to "higher stage tiers, better
  rewards" — keyed off roster *size*, not highest rarity owned, since
  the starter six being pre-owned Legendaries would have made a
  rarity-keyed gate a no-op from turn one (see Systems item 13 for the
  full reasoning).

**Epic-tier champions — built**: Bastian/Morwen/Talon/Raze, one per
faction (see Systems item 1). The rarity ladder is now fully populated
(Common through Legendary at every rarity a summon can roll), every
faction has 3+ members, and both faction bonus tiers are reachable
everywhere. Legendary summons still always refund as a duplicate —
every Legendary is a pre-owned starter — which is expected, not a bug;
revisit only if a summonable Legendary is ever added on purpose. The
champion-acquisition end cycle the user asked for is now fully built
end to end: summon a champion → grow rarity/faction bonuses → raise
Roster Tier → push further on every infinite ladder → better rewards →
more Summon Shards → summon again.

Note: "Rework node" turned out to mean gear substat/set/stat-value
rerolls, not a champion build-choice system — champions still have
zero build choices, so a champion-facing Rework is not on the table
until one exists (see the champion build-choice bullet below).

**New-game onboarding / Campaign Chapter 1 tutorial ladder — built.**
The user's framing: a new save opens with a weak starter squad, learns
the game across a 12-15-stage first chapter, earns two guaranteed
champions along the way, hits a real plateau that forces a detour into
whichever mode just unlocked, and that farm-and-push cycle repeats for
every subsequent plateau — see "New-game onboarding" under Content
framework above for the full mechanism (`CAMPAIGN_CH1_STAGES`,
`CAMPAIGN_STAGE_CHAMPION_REWARDS`, the retuned `CHAMPION_TRIAL`
baseline, the new `CONTENT_UNLOCKS` sequencing).

**Chapters 2-11 difficulty seam — retuned.** Follow-up to the Chapter 1
rework above: the old Chapters 1-10 (now displaying as 2-11) were still
tuned for the pre-rework all-Legendary squad and had gone flat-trivial
for 7 straight chapters before falling off a cliff — see "Chapters
2-11 difficulty seam — retuned" under Content framework above for the
full probe history and final numbers.

Explicitly still open, per the user's own framing ("only then decide
whether Chapters 2-10 get the same multi-stage treatment or stay as
capstone fights between ladders") — do not start that work without
being asked.

Longer-term, deferred until asked for:
- A **Charm Upgrade** to pair with the new Charm Reforge — gear has
  both Upgrade (level a piece's rolled values) and Reforge; charms
  only have Reforge + Salvage so far. Revisit if charm power creep
  becomes an issue.
- A champion build-choice system (talent variants, stat-path choices,
  etc.) — nothing like this exists yet; a champion-facing Rework mode
  would need it first.
