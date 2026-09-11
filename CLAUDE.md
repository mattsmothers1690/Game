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
| `machineborn_gold_v1` | Gold currency (a number) — Guild Boss chests (Main Boss) / champion-leveling only | Persistent |
| `machineborn_shards_v1` | Shard currency (a number) — Guild Boss chests (Main Boss) / level-cap only | Persistent |
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
| `machineborn_auto_config_v1` | `{ championId: { active1, active2, active2First } }` — per-champion Auto Battle skill priority/enable config | Persistent |
| `machineborn_last_squad_v1` | `{ encounterId: [championId, ...] }` — the exact squad last fielded per mode, defaulted back on the next team-select for that mode | Persistent |

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
    writes (all 20: equipment/gear/charm inventories, every currency,
    champion progress/roster, level cap, stage progress, the mid-battle
    save, the auto-battle skill-priority config, the last-fielded-squad
    memory) in one list, referenced
    by each key's own `_KEY` constant
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
15. **Auto Battle** (`autoBattleOn`/`autoChooseKindAndTarget`/
    `runAutoTurn`): a Raid-style auto-play toggle scoped to this game's
    basic/active1/active2 kit (there's no third active here, so
    "priority" only ever orders two actives against each other, not
    three). Deliberately reuses the exact same
    `playerChooseAbility`/`playerChooseTarget` pipeline manual clicks
    go through — auto only decides *which* skill and *which* target,
    every mechanic downstream (cooldowns, Silence, extra turns) is
    identical to manual play, nothing was reimplemented in parallel.
    - **Standard priority is highest-slot-first** (Active2 > Active1 >
      Basic, the convention the user asked for): `autoChooseKindAndTarget`
      walks `availableActiveKinds(actor)` (the same off-cooldown/
      not-Silenced filter Grand Arena's AI already used) in priority
      order and falls back to Basic if nothing usable is enabled.
    - **Per-champion config** (`AUTO_CONFIG_KEY`,
      `machineborn_auto_config_v1`, `{ [championId]: { active1,
      active2, active2First } }`) lets either active be disabled
      entirely (kept in reserve, never auto-spent) or the priority
      order flipped to Active1-first — same "toggle off / reorder"
      controls Raid's own skill-priority screen offers, via the
      in-battle ⚙ PRIORITY panel (`renderAutoPriorityList`,
      `#autoPriorityOverlay`). Config keys are read with `!== false`/
      `=== false` checks rather than requiring all three keys present,
      so a partial stored object (only one key ever touched) still
      falls back to the documented defaults for the rest.
    - **BASIC ONLY** (`autoBasicOnly`) is a separate global override
      button, not a per-champion setting — the user's own framing was
      "just get through the wave and save abilities," a blunt
      fight-level choice, not a loadout preference — so it skips the
      whole priority branch and always picks Basic, regardless of any
      champion's individual config.
    - **Targeting** reuses `lowestHpAlly` (already generic — despite
      the name, it just finds lowest-HP%-of-list, used elsewhere for
      both ally and foe lists) against `aliveTeam()` for
      `targetType: 'ally'` actives (heals/buffs/shields) or
      `aliveEnemies()` otherwise — the same "support the neediest
      ally / finish the squishiest foe" heuristic Grand Arena's Stall/
      Aggro archetypes already use, not a new targeting concept.
    - **Deliberately NOT persisted**: `autoBattleOn`/`autoBasicOnly`
      reset to off on every `newBattle`/`resumeBattle` by default — same
      as Raid's own Auto toggle, so a forgotten toggle from a prior
      fight can never silently burn a real one. Only the skill-priority/
      enable config persists (account-wide, like equipped gear), and
      it's included in `allSaveKeys()` for New Game reset.
    - Manual skill/target clicks are ignored (`if (autoBattleOn)
      return;` guards on the `#skillRow`/`#enemyList`/`#teamList`
      click handlers) while Auto is on, rather than letting both
      inputs race — toggle Auto off to act manually again, matching
      Raid's own behavior.
    - **Team-select AUTO toggle** (`#teamSelectAutoBtn`/
      `pendingAutoStart`): the one deliberate exception to "reset to off
      every battle" above — the user asked to be able to start a fight
      already in Auto mode instead of always having to toggle it on
      again after Begin Battle. `pendingAutoStart` is a team-select-only
      flag (reset to `false` every time `openTeamSelect` runs, so it
      never leaks from one encounter's setup screen into the next) that
      flows through `beginEncounter`/`newBattle` as a `startInAuto`
      argument; `newBattle` sets `autoBattleOn = !!startInAuto` instead
      of unconditionally `false`. `resumeBattle` is untouched (still
      always starts non-auto) — this toggle is about *starting* a fresh
      fight in Auto, not resuming one.
16. **Team-select defaults: highest tier owned + remembers last squad**
    (`defaultSquadFor`, `machineborn_last_squad_v1`): the user asked for
    two related things — default to the best champions you actually
    own, and remember what you fielded last time in that same mode.
    `openTeamSelect` now calls `defaultSquadFor(encounterId, stage)`
    instead of blindly pre-selecting `ENCOUNTERS[id].teamIds` filtered
    by ownership. `defaultSquadFor` first checks
    `lastSquad[encounterId]` (loaded from `machineborn_last_squad_v1`) —
    if every member is still owned AND the remembered squad's length
    still matches the mode's current `squadSizeFor` (a remembered 3v3
    Chapter 1 squad does NOT carry over onto a 4v4 Chapter 2+ stage), it
    wins outright. Otherwise it falls back to the highest-tier-owned
    default: every owned champion sorted by rarity (`RARITY_ORDER`,
    Legendary first) then by level, taking the top N for the mode's
    squad size — reusing the same rarity scale gear/champions already
    share rather than inventing a new "tier" concept. `lastSquad` is
    written once, in `beginEncounter` (the one choke point every mode's
    Begin Battle click already funnels through), so it captures the
    squad that actually fought, not just whatever was clicked and then
    abandoned. `squadSizeFor` gained a second `stage` argument to make
    Campaign's per-chapter squad size (see the 4v4/3-wave restructure
    under "Content framework" below) resolvable at team-select time.
    `LAST_SQUAD_KEY` is included in `allSaveKeys()` for New Game reset.
17. **Auto-Equip** (`bestAutoEquipTarget`/`autoEquipGear`, victory-
    screen `#autoEquipBtn`): offered right after a gear drop, scoped to
    `currentTeamIds` (the squad that just fought) per the user's own
    framing — "one of the champs you just used." Prefers an empty slot
    on one of them first (tie-broken toward whoever already has other
    pieces of the drop's set equipped, for set-bonus continuity via
    `equippedSetCounts`); only once every candidate already has that
    slot filled does it fall back to an upgrade comparison
    (`gearInstancePower` — main stat + substats, level-adjusted via
    `effectiveStatValue` — summed as a simple total-roll-value heuristic,
    not a full simulated-DPS model, consistent with this project's
    general "the obvious heuristic, not an elaborate one" bias). A
    reforged/reworked piece never displaces something better: the gain
    must be strictly positive, with a small bonus weight added only
    when the swap would newly reach a `SET_TIERS` threshold (2pc/4pc/
    6pc). Manual Armory equipping is completely unaffected — Auto-Equip
    is an additional, optional shortcut on the victory screen, not a
    replacement for `cycleGearSlot`.
18. **Main Boss: Guild Boss redesign** (`GUILD_BOSS_WAVE`,
    `checkGuildBossChests`/`grantGuildBossChest`): replaced the old
    generic 3v3 squad (`CHAMPION_TRIAL`, an infinite ladder like every
    other mode) with a single, deliberately very tanky boss per level,
    per the user's explicit "guild boss" framing — the point is a fight
    nobody one-shots, with a reward chest unlocking at each quarter of
    its health lost (75%/50%/25%/0%, `GUILD_BOSS_CHEST_FRACTIONS`) and
    full defeat unlocking the next level. Deliberately reuses the
    existing infinite-`stageProgress` ladder machinery wholesale rather
    than inventing a parallel "level" system — for Main Boss specifically,
    a stage *is* a level; `getSelectedStage`/`setSelectedStage`/
    `maxSelectableStage`/roster-tier gating all keep working unmodified.
    `mainbossStageLabel` (`ENCOUNTERS.mainboss.stageLabelFor`) renders it
    as "Guild Boss — Level N" instead of "Stage N", the same hook
    pattern Campaign/Grand Arena already use for their own custom
    labels. Chests are granted the instant their threshold is crossed,
    mid-fight (checked every `checkBattleEnd`, not held back for a full
    clear) — a loss still keeps whatever chests were earned that
    attempt, which is the actual point: farmable progress even on a
    fight you don't finish. `guildBossChestReward(level, tierIndex)`
    ramps pay along both axes the user asked for — later chests within
    one level's fight pay more (`tierWeight` 0.15/0.2/0.25/0.4), and
    every level's chests all pay more than the last level's
    (`base = 25 + level*15`) — Gold/XP every chest, Shards only on the
    4th/defeat chest (keeps the level-cap currency scarce), Summon
    Shards from the 3rd chest onward. Main Boss's Gold/XP/Shard/Summon
    Shard identity moved entirely off the generic post-victory
    `ENCOUNTER_*_REWARD` tables (would double-pay otherwise) onto this
    chest mechanic — the victory-screen reward line for Main Boss no
    longer lists these (mid-fight `logLine` calls announce each chest
    instead), a known, deliberate UX tradeoff (see "Next up"). Resuming
    a saved Guild Boss fight (`resumeBattle`) infers which chests were
    already claimed pre-save from the boss's restored hp ratio, so a
    reload-mid-fight can never re-grant (or under-grant) a threshold.
    `isBoss: true` on `GUILD_BOSS_WAVE` reuses the existing devour-a-
    debuff-for-Fury mechanic (`hollowKingMechanic`) like every other
    named boss in the game — no parallel mechanic invented. Numbers
    (900 hp base, 40 atk/30 def) are a first pass sized so a fresh
    starter squad takes many turns rather than either one-shotting or
    being one-shot — not yet probed across many levels/gear tiers, same
    "expect iteration" caveat as every other hand-tuned encounter in
    this file.

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
| **Campaign** (`campaign`) | 3v3 (Chapter 1 only) / 4v4 × 3 waves (Chapters 2-11) | 132 fixed hand-authored stages (`CAMPAIGN_CHAPTERS` = `CAMPAIGN_CH1_STAGES` (12) + 10 chapters × 12 stages each via the `buildChapterArc` generator — see "New-game onboarding" and "Campaign 4v4/3-wave restructure" below) — **not** the infinite ladder, no stat compounding, replaying an old stage is always the same fight | Always open | Scrap + Gold only (bootstrap) |
| **Tower** (`tower`) | 5v5 | Infinite stage ladder | Stage 7 | Gear (guaranteed) + Gear Rework Catalysts (rare, rides gear drops) + Scrap |
| **Faction Wars** (`faction`) | 4v4 × 3 waves | Infinite stage ladder | Stage 10 | Charms (rolled chance) + Charm Dust (guaranteed) |
| **Main Boss** (`mainboss`) | 3v3 vs. 1 Guild Boss | "Levels" (reuses the infinite-ladder `stageProgress` mechanism — a level IS a stage) | Stage 4 | Gold + XP + Shards + Summon Shards, paid out via 4 chests unlocked at 75%/50%/25%/0% of the boss's health, ramping in value per chest and per level (see "Main Boss: Guild Boss redesign" below) |
| **Dungeon** (`dungeon`) | 4v4 | Infinite stage ladder | Stage 132 (Campaign complete) | Ascension Cores only |
| **Galactic War** (`galactic`) | 5v5 × 5 waves | Infinite stage ladder, no retreat between waves within one attempt | Stage 132 (same threshold as Dungeon — a "you finished Campaign" bonus node, not a fifth step in the unlock sequence) | Bulk Scrap + Gold + XP only — no gear/charms/shards/catalysts/Ascension Cores |
| **Grand Arena** (`grandarena`) | 3v3 vs. one of 4 AI archetype squads | Infinite stage ladder, stage number cycles through the 4 archetypes (`(stage - 1) % 4`) forever | Stage 132 (alongside Dungeon/Galactic War) | Arena Medals only — spent on Arena Pulls (Vanguard-set gear), never dropped directly |

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
  `fixedChapters` entries prepended to the `CAMPAIGN_CHAPTERS` array
  (now 132 stages total, once Chapters 2-11 each became their own
  12-stage 4v4/3-wave arc — see "Campaign 4v4/3-wave restructure" and
  "Chapters 2-11 mini-boss/boss
  expansion" below) — this needed zero changes to
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
- Chapters 2-11 (the old Chapters 1-10) were first retuned in place
  (still one hand-authored fight per chapter), then — once the user
  asked for mini-bosses/bosses in the Campaign — each expanded into its
  own 3-stage regular/mini-boss/boss trio, the same shape Chapter 1
  already uses at Stage 9/12 one tier further out. See "Chapters 2-11
  mini-boss/boss expansion" below for the full mechanism; the retune
  history that preceded it is folded into that same section since the
  expansion's new numbers are built directly on top of it.

**Chapters 2-11 mini-boss/boss expansion.** Before this pass, Chapters
2-11 were still single fights (the original CH1-CH10 numbers,
hand-tuned for a level-1 all-Legendary 3-champion squad, then retuned
once for the post-Chapter-1 starter squad — see the retune history
below). The user then asked for a mini-boss at each chapter's halfway
point and a boss at the end, and confirmed (via `AskUserQuestion`) the
larger of two scopes on offer: split each of Chapters 2-11 into its
own multi-stage mini-ladder like Chapter 1, rather than just re-tagging
each chapter's existing single 3-enemy squad. This is the exact "give
Chapters 2-10 the multi-stage treatment" decision earlier passes of
this doc had flagged as explicitly deferred — now done.
- **Structure**: each old chapter (`CAMPAIGN_CH1`..`CAMPAIGN_CH10`,
  displaying as Chapter 2-11) became 3 stages — `CAMPAIGN_CH{n}A`
  (regular warm-up), `CAMPAIGN_CH{n}B` (mini-boss at the chapter's
  halfway point, 2 regular units + 1 named tougher unit, no `isBoss`),
  `CAMPAIGN_CH{n}C` (2 elite units + 1 named boss, `isBoss: true`).
  `CAMPAIGN_CHAPTER_LENGTHS = [12, 3,3,3,3,3,3,3,3,3,3]` (Chapter 1's
  12 tutorial stages, then 3 per chapter after it) replaces the old
  binary "stage <= 12 or a flat Chapter N" label logic with a general
  `campaignChapterInfo(stage)` helper (walks the lengths array to find
  which chapter and which sub-stage) that both `campaignStageLabel`
  ("Chapter N - Stage M/T", now uniform across every chapter including
  Chapter 1, where before only Chapter 1 showed a sub-stage count) and
  `campaignClearMessage` (sub-stage-relative "Stage M cleared - Stage
  M+1 unlocked!" within a chapter, "Chapter N cleared - Chapter N+1
  unlocked!" at a chapter boundary) now read off directly. Total
  Campaign length at this point: 12 + 10×3 = 42 stages (up from 22) —
  later expanded again to 132 by the 4v4/3-wave restructure below;
  `CAMPAIGN_CHAPTERS.length` still drives `CONTENT_UNLOCKS.dungeon/
  galactic/grandarena` automatically either way, no separate change
  needed there.
- **The two chapters that already had a real boss** (Chapter 9's
  Campaign Warlord, Chapter 11's The Herald) keep that exact,
  previously-validated boss stage **completely unchanged** as their new
  Stage C — reusing already-probed numbers rather than re-deriving them
  preserves every prior calibration point, especially the two hardest
  fights in the game. Their two new stages (A/B) scale *down* from
  their own existing "Elite Guard"/"Herald Guard" numbers (there was no
  separate weaker regular unit to derive from in these two chapters to
  begin with — every unit already read as elite). Their new mid-chapter
  mini-bosses are original: "Ashen Enforcer" (Chapter 9) and "Hollow
  Acolyte" (Chapter 11).
- **The other 8 chapters had no boss unit at all** (just 2 regular + 1
  "special" unit, e.g. Bramble Scout ×2 + Bramble Brute) — each gained
  a genuine new named boss: Grunt Warchief (Ch.2), Bramble Ancient
  (Ch.3), Fen Matriarch (Ch.4), Iron Centurion (Ch.5), Ashen High
  Priest (Ch.6 — a deliberate tie to the Ashen Cult's rise already in
  the lore for this chapter), Steel Warmaster (Ch.7), Dusk Reaver
  (Ch.8), and The Devourer (Ch.10 — a deliberate tie to the Hollow
  King's hunger, right before Chapter 11's Herald). Each chapter's
  original "special" unit (Bramble Brute, Fen Warden, Iron Archer,
  Ashen Zealot, Steel Marksman, Dusk Overseer, Void Harbinger) became
  that chapter's Stage B mini-boss, scaled up rather than replaced —
  it already read as a cut above its squadmates, so it earned the slot
  rather than needing a new identity.
- **Numbers use a top-down target curve, not a bottom-up derivation.**
  An early attempt derived each new boss purely from its own chapter's
  existing regular/special units in isolation, and produced a
  non-monotonic climb that briefly out-statted the already-validated
  Chapter 9/11 bosses (Chapter 8's derived boss came out higher-HP than
  Chapter 9's own Campaign Warlord) — backwards, since Chapter 9 is
  supposed to still be the harder wall. Fixed by setting one explicit
  target hp/atk/def curve across all 10 new-or-kept bosses first
  (climbing gently Chapter 2→8, holding Chapter 9's real numbers as the
  wall, dipping at Chapter 10 for relief, holding Chapter 11's real
  numbers as the hardest fight), then deriving each chapter's "elite"
  support pair, mini-boss, and regular stage down from *that* target
  via one consistent set of ratios (elite ≈0.58/0.80/1.0 × boss's
  hp/atk/def; mini-boss ≈0.90 × elite; Stage B's support pair ≈0.70 ×
  elite; Stage A's regulars ≈0.55 × elite) — not from each chapter's
  own pre-existing units. Reused abilities/debuffs (Poison, Bleed,
  Defense Down, Speed Down, Resistance Down) stayed on whichever new
  unit descends from the original template that carried them.
- **Verified via spot-probes only** (`probe_ch_bosses.js`,
  `test_boss_flags.js`, `test_chapter_expansion.js`), not an exhaustive
  8-trial-per-stage sweep of all 30 new stages — that would be
  impractical at this scale. Confirmed: Chapters 2/6's regular-to-boss
  climb reads as expected warm-up/build territory (8/8 at this probe's
  squad power, consistent with Chapters 2-8's already-established
  easy/building character), Chapter 9's boss stage is untouched and
  still hits its historical ~0/8 wall, Chapter 10's new boss reads as a
  real-but-passable relief step (~7/8), and Chapter 11's finale is
  untouched and still hits its historical ~2/8. Also confirmed via the
  `window.__debug` hook that mini-boss units never carry `isBoss` and
  every chapter's actual boss does, matching the design intent exactly
  (not just a stat bump wearing a boss-sounding name). Treat this as a
  first pass on 30 brand-new stages, same as everything else in this
  file — expect iteration once real playtesting (not a naive
  first-target-click auto-bot) exercises it.
- `CAMPAIGN_STORY`/`CAMPAIGN_STORY_AFTER` (see "Lore & Narrative"
  below) were rewritten in full for this 42-stage range — every
  mini-boss and boss above gets its own setup line and payoff line,
  not a reused chapter-level summary. **Now stale/incomplete**: the
  4v4/3-wave restructure below expanded Campaign again, to 132 stages,
  and the lore tables were NOT re-extended to match — stages 1-42 still
  show their blurb, stages 43-132 show none (the same graceful "no
  blurb" fallback every non-Campaign encounter already uses, so this is
  a missing-content gap, not a bug). The Lore & Narrative section's own
  stage-number citations (Act II/III ranges, specific stage numbers)
  also still describe the OLD 1-42 numbering and have not been
  re-mapped onto the new 1-132 range. Revisit if the lore beats matter
  enough to extend — see "Next up".

**Campaign 4v4/3-wave restructure.** The user asked, in one further
follow-up, for Campaign battles generally to be 4v4 with 3 waves per
stage, with a mini-boss stage (preceded by warm-up waves) at each
chapter's Stage 6 and a boss stage (alone, no pre-waves) at Stage 12 -
applied, per the user's explicit choice between offered scopes, to
*every* chapter rather than just reformatting the existing single-fight
stages. This is layered on top of the mini-boss/boss expansion above,
not a replacement for it: the CAMPAIGN_CH*A/B/C arrays (30 chapters'
worth of regular/mini-boss/boss templates) stay exactly as documented
above and are now the *input* to a new generator rather than the final
stage list themselves.
- **Chapter 1 is deliberately excluded** and stays 3v3/single-wave,
  unchanged. Requiring 4 champions before Chapter 1 Stage 6 (the exact
  stage that grants Bastian, the roster's 4th member) would brick new-
  game onboarding, since a fresh save owns only the 3 starters until
  then. `squadSizeFor('campaign', stage)` special-cases this: 3 for
  stage ≤ `CAMPAIGN_CH1_STAGE_COUNT` (12), 4 above it. By Chapter 2
  (Stage 13) the roster already has 5 owned champions (3 starters +
  Bastian + Vara), so 4v4 is always fieldable from there on.
- **`buildChapterArc(chId, regularArr, miniBossArr, bossArr)`** expands
  each old chapter's regular(A)/mini-boss(B)/boss(C) trio into a fresh
  12-stage arc, generated at script-init time rather than hand-authored
  (120 new stages across 10 chapters would be impractical to author by
  hand, consistent with the top-down-curve reasoning already used for
  the mini-boss/boss numbers themselves): Stages 1-5 are a regular ramp
  (4v4, 3 waves/stage, `regular` unit scaled 1.0x-1.35x via
  `scaleUnitStats`); Stage 6 is the mini-boss (4v4, 3 waves - 2 warm-up
  waves plus a finale wave where the named mini-boss joins 3 regulars,
  the "pre-waves" the user asked for); Stages 7-11 are an elite ramp
  (4v4, 3 waves/stage, `elite` unit scaled further up); Stage 12 is the
  boss alone (4v4, a single wave with no pre-waves - 3 elite escorts
  plus the named boss, stats untouched from the original CH*C arrays,
  including both already-validated real bosses, Campaign Warlord and
  The Herald). `waveOf4` produces 4 slightly speed-staggered copies of
  a template per wave. Every named unit/number is reused verbatim from
  the CAMPAIGN_CH*A/B/C arrays - only the ramp stages and the
  wave/squad-count reshaping are generated.
- **`wavesForBattle` now supports multi-wave `fixedChapters` entries**:
  a stage entry is either a flat template array (one wave, still true
  for all of Chapter 1 and every chapter's Stage 12) or `{ waves: [...] }`
  (Stages 1-11 of Chapters 2-11) - resolved generically, no per-mode
  branching needed since the underlying wave-advance loop in
  `checkBattleEnd` was already generic (the same mechanism Faction
  Wars/Tower/Galactic War already use for their own multi-wave configs).
- **`CAMPAIGN_CHAPTER_LENGTHS`** is now `[12, 12, 12, ..., 12]` (Chapter
  1's 12 plus 12 for each of the 10 chapters after it) instead of
  `[12, 3, 3, ..., 3]` - `campaignChapterInfo`/`campaignStageLabel`/
  `campaignClearMessage` needed zero code changes, since they already
  read chapter boundaries generically off this array. Total Campaign
  length: 12 + 10×12 = **132 stages** (up from 42).
  `CAMPAIGN_CHAPTERS.length` still drives `CONTENT_UNLOCKS.dungeon/
  galactic/grandarena` automatically - they now unlock at Stage 132.
- **Verified via `test_five_features.js`** (scratchpad, not committed):
  confirms `CAMPAIGN_CHAPTER_LENGTHS`/total stage count, that Chapter 1
  stays 3v3 while Chapter 2+ is 4v4, that a Chapter 2 regular stage is a
  3-wave/4-enemy fight, that Chapter 2's boss stage (global Stage 24) is
  a single wave with Grunt Warchief present and `isBoss:true`, and that
  Chapter 2's mini-boss stage (global Stage 18) is 3 waves with Grunt
  Brute in the finale wave without `isBoss`. Not an exhaustive sweep of
  all 120 new stages, matching this file's established testing
  philosophy for large generated content batches.

Implementation notes:
- **Fixed vs. infinite vs. cycling**: `ENCOUNTERS[id].fixedChapters`
  (an array of chapter template arrays) marks Campaign as non-scaling —
  `spawnWave` passes stage `1` to `scaledEnemyTemplate` for such
  encounters instead of the real stage, so a chapter's authored numbers
  ARE its difficulty, forever. `getSelectedStage`/`setSelectedStage`
  both cap at `fixedChapters.length` (now 132) so there's no "Stage 133"
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
  devour-a-debuff-for-Fury mechanic (`hollowKingMechanic` — triggers
  off `actor.isBoss` generically, not literally the Hollow King itself,
  but the name was never a placeholder: see "Lore & Narrative" below —
  every `isBoss` unit devouring a debuff for Fury is now canonically
  the Hollow King's hunger expressing itself through whichever body is
  currently host to it) rather than a parallel one.
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

## Lore & Narrative

The user asked for backstory explaining why the game's factions are
"battling and fighting each other," to use in the Campaign. Full text
lives in a published Artifact ("The Unclaimed Line" —
https://claude.ai/code/artifact/05ad1907-8c60-4e1a-ac27-995b6b26d66c);
this section is the canonical summary for future dev work, since
CLAUDE.md — not an external artifact link — is this project's actual
source of truth.

**Stage-number staleness note**: everything below (the three-act
structure, `CAMPAIGN_STORY`/`CAMPAIGN_STORY_AFTER`, specific stage
citations like "Stage 42") was written when Campaign was 42 stages
total. The Campaign 4v4/3-wave restructure (see "Content framework"
above) later expanded Campaign to 132 stages by giving Chapters 2-11
12 stages each instead of 3 - the narrative beats and named enemies
below are all still accurate and still occur in the same order, but
their specific stage-number citations now describe the OLD 1-42
numbering, not the current 1-132 one, and the in-game story blurbs
(`CAMPAIGN_STORY`) stop showing entirely past (old) Stage 42 - i.e.
roughly a quarter of the way into the new Campaign. Not re-mapped as
part of that restructure since it was scoped to battle format, not
lore; revisit if the lore beats matter enough to extend across the
full 132 stages.

- **The four playable factions each have a home, a creed, and an old
  grudge against at least one other faction** — texture explaining
  "battling and fighting each other" as history/tension rather than
  literal Campaign combat (the squad is drawn from all four factions
  cooperating, never fighting itself):
  - **Ironclad** (Vara, Wren, Bastian) — the Ashenwall March, a
    fortified border nation built on "hold the line" as both military
    doctrine and personality. Old wound: turned away Blightkin plague
    refugees generations ago under quarantine law; the law was proven
    right, Blightkin never forgave it anyway.
  - **Blightkin** (Mire, Nyx, Morwen) — the Rotmire Fens, survivors of
    a plague they didn't cure but bonded to their own blood as a
    weapon rather than an affliction. Distrusts Ironclad specifically
    (see above); trades freely with Stormcallers, warily with
    Deathmark.
  - **Stormcallers** (Kestrel, Zephyr, Squall, Talon) — the Skyreach
    Steppes, a nomadic sky-nation of favor-and-grudge bookkeeping,
    driven off richer lowland territory generations ago by an
    Ironclad-Deathmark border dispute neither of those two even
    remembers clearly. Deals fairly with everyone, trusts no one
    completely.
  - **Deathmark** (Rook, Fang, Vex, Raze) — the Marked City, a
    deliberately unmappable order of bounty-hunters/executioners
    ("the Ledger") founded to hunt war criminals for the other three
    factions, never fully answerable to any of them since. Universally
    hired, universally distrusted.
- **The antagonist is the Hollow King** — resolves what
  `hollowKingMechanic` (every `isBoss` unit devouring a stacked debuff
  for Fury) always implied but never had lore behind: an ancient,
  possibly-never-human hunger sealed beneath the contested, officially
  "neutral" borderland none of the four factions will settle — the
  real reason that land stayed unclaimed for longer than any explained
  tradition. It feeds on affliction (the literal in-fiction reason for
  the Fury mechanic) and on the four factions' compounding grudges
  alike — division is fuel, which is why it surfaced exactly here.
- **The 42-stage Campaign maps onto a three-act structure**, using
  existing enemy names as-is wherever they predate the mini-boss/boss
  expansion (no renaming needed, every one already fit) and new named
  mini-bosses/bosses for every chapter that gained one (see "Chapters
  2-11 mini-boss/boss expansion" above for exactly which):
  - **Act I — Chapter 1 (Stages 1-12), "The Unclaimed Line":** the
    tutorial ladder's own arc — a mixed Stormcaller/Deathmark patrol
    (Zephyr/Fang/Squall) investigates wrong wildlife and banditry,
    gains Bastian (Stage 6) and Vara (Stage 8) as Ironclad
    reinforcements in-fiction, hits Stage 9's wall as the first
    visibly-corrupted warband, and Stage 12's Chapter Warlord as the
    first named enemy showing outside direction.
  - **Act II — Chapters 2-8 (Stages 13-33), "Four Borders, One Wound":**
    the same pattern erupts on all four factions' borders
    simultaneously, forcing the first-ever mixed joint task-forces
    (the in-fiction reason any 3-champion squad from any faction mix
    is normal from here on) as the threat escalates from wildlife/
    banditry into the openly-worshipping Ashen Cult. Each chapter now
    has its own mini-boss/boss beat: Grunt Warchief (Ch.2), Bramble
    Ancient (Ch.3), Fen Matriarch (Ch.4), Iron Centurion (Ch.5), Ashen
    High Priest (Ch.6 — the Cult's first named leader), Steel Warmaster
    (Ch.7), Dusk Reaver (Ch.8).
  - **Act III — Chapters 9-11 (Stages 34-42), "The Mask Comes Off":**
    Chapter 9's Campaign Warlord (Stage 36, preceded by a second, worse
    Ashen Enforcer at Stage 35) unifies the scattered warbands — the
    real wall; Chapter 10's Void Marauders/Harbinger/The Devourer
    (Stage 39) reveal cultists as conduits rather than people; Chapter
    11's Hollow Acolyte (Stage 41) and The Herald (Stage 42, Campaign's
    true finale) is the Hollow King's first direct avatar in the world,
    not the Hollow King itself — deliberately leaves the actual Hollow
    King unfought, open for future content.
  - **After the Herald:** Dungeon = the vaults the Herald's breach
    uncovered (the hunger's oldest leftovers, a debuff-mitigation test
    rather than a stat check — ties into Dungeon's existing design
    identity, not a new one). Galactic War = the bulk mop-up of
    warbands that don't vanish with one boss fight (ties into its
    existing "big generic farm, no named villain" identity). Grand
    Arena = the four factions sparring against each other's best
    on purpose, non-lethally, because the wartime alliance needs a
    reason to outlive the war (ties into its existing "vs. AI
    archetype squads" identity as friendly, not hostile, competition).
- **Surfaced in-game as a per-stage story blurb** (`CAMPAIGN_STORY`,
  keyed 1-42 by the same flat stage index everything else in Campaign
  uses): one short italic line rendered in `#teamSelectStory` at
  team-select, right before the fight it precedes — wired into
  `renderTeamSelect` (`pendingEncounterId === 'campaign' ?
  CAMPAIGN_STORY[pendingStage] : null`, hidden entirely on every other
  encounter). Every mini-boss/boss introduced by the expansion above
  gets its own setup line at its stage, not a reused chapter-level
  summary — e.g. Stage 14 (Chapter 2's mini-boss) reads "One raider
  doesn't break like the rest...", Stage 15 (its boss) reads "Whatever
  answers to 'Warchief' out here commands the whole warband...". A
  matching `CAMPAIGN_STORY_AFTER` table pays off each of those lines on
  the victory screen (`#resultStory`, wired into `endBattle`) — written
  as a direct continuation of the matching `CAMPAIGN_STORY` entry, not
  a restatement of it (Stage 1: patrol dispatched → rats scatter, cause
  unclear; Stage 42: the Herald named → the Herald shatters, the Hollow
  King still out there). Shown only on an actual `won` Campaign result
  — a `DEFEAT` never earns the consequence of a win it didn't get, and
  every non-Campaign encounter still shows neither table. Both tables
  were rewritten in full when Chapters 2-11 expanded from 1 stage each
  to 3 — the old 22-entry versions are gone, not left stale alongside
  the new ones. A deliberately light touch throughout: one line per
  stage, no dialogue system, no branching, no new screen — the existing
  team-select/victory moments already fire once per stage and had the
  room. A full Codex/lore screen for the faction dossiers themselves is
  still not built — revisit only if asked for a deeper reference than
  the Artifact already provides.

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
`CAMPAIGN_STAGE_CHAMPION_REWARDS`, the then-current `CHAMPION_TRIAL`
baseline — since replaced by the Guild Boss redesign, see Systems item
18 — and the new `CONTENT_UNLOCKS` sequencing).

**Chapters 2-11 mini-boss/boss expansion — built.** Follow-up to the
Chapter 1 rework above, in two steps: first a straight difficulty
retune (Chapters 2-11 were still tuned for the pre-rework all-Legendary
squad and had gone flat-trivial for 7 straight chapters before falling
off a cliff), then — once the user asked for a mini-boss at each
chapter's halfway point and a boss at the end — a full expansion of
each chapter from 1 fight into its own 3-stage regular/mini-boss/boss
mini-ladder, the exact "give Chapters 2-10 the multi-stage treatment"
decision this doc had previously flagged as deferred. See "Chapters
2-11 mini-boss/boss expansion" under Content framework above for the
full mechanism, naming, and probe history. Campaign was 42 stages
total at this point (was 22) — later expanded again to 132 by the
4v4/3-wave restructure below.

**Faction/Hollow King lore — written and now walked through in the
Campaign.** The user asked for backstory explaining the factions
"battling and fighting each other," then asked for the Campaign itself
to walk the player through it as a story-driven campaign — see "Lore &
Narrative" above for the canonical summary (including the
`CAMPAIGN_STORY` in-game integration) and the published Artifact for
the full-length version. A dedicated Codex/lore screen for the full
faction dossiers is still not built — revisit only if asked for a
deeper in-game reference than the current one-line-per-stage treatment
plus the Artifact.

**Five-item follow-up request — built.** In one message the user asked
for: (1) team-select defaulting to the highest-tier champions owned and
remembering the last squad used per mode; (2) an Auto-Equip button on a
gear drop; (3) an Auto Battle toggle at the team-select/setup screen so
a fight can start already in Auto; (4) Main Boss redesigned into a
"guild boss" — one tough boss, not a 1-shot, with a reward chest per
quarter of its health and a "level" that unlocks on full defeat,
ramping rewards per chest and per level; (5) Campaign battles generally
restructured to 4v4 with 3 waves, a mini-boss stage with pre-waves at
each chapter's Stage 6, and the boss alone (no pre-waves) at Stage 12.
All five are built — see Systems items 16-18 and the Campaign
4v4/3-wave restructure under "Content framework" above for the full
mechanisms. Two scope calls made along the way, both surfaced to the
user rather than assumed silently:
- Item 5 was confirmed, via `AskUserQuestion`, to mean restructuring
  *every* chapter (not just Chapter 1, and not just a battle-format
  swap on the existing 3-stage-per-chapter shape) — the larger of three
  offered scopes.
- Chapter 1 itself was deliberately kept OUT of that restructure and
  stays 3v3/single-wave — requiring 4 champions before Chapter 1 Stage
  6 (the stage that grants Bastian, the roster's 4th member) would
  brick new-game onboarding for a fresh save that only owns 3 starters.
  This is a judgment call made during implementation, not something the
  user was asked about directly — flagged here for visibility.

Known gaps/tradeoffs from this pass, left for a future iteration:
- `CAMPAIGN_STORY`/`CAMPAIGN_STORY_AFTER` and the Lore & Narrative
  section's own stage-number citations were NOT re-extended/re-mapped
  onto the new 132-stage Campaign (they still reflect the old 1-42
  range) — see the staleness note at the top of "Lore & Narrative".
- Main Boss's victory-screen reward line no longer lists Gold/XP/
  Shards/Summon Shards (those are announced via mid-fight `logLine`
  chest messages instead, not surfaced in the final summary) — a
  deliberate but not fully polished consequence of moving that reward
  identity onto the chest mechanic.
- `GUILD_BOSS_WAVE`'s stats (900 hp, 40 atk/30 def) and
  `guildBossChestReward`'s payout curve are a first pass, probed only
  at Level 1 — expect retuning once real play exercises higher levels
  against leveled/geared squads.
- Several pre-existing scratchpad Playwright tests (not committed —
  see "Testing methodology") hardcode the old 42-stage Campaign total
  for unlock-gating seeds (`galactic_war`, `grand_arena`,
  `roster_tier`, `dungeon_ascension`, `ch1_rework`,
  `campaign_expand_basic`) and are now stale, not failing due to a real
  regression — same "stale test, not a regression" convention as
  always, just not all individually fixed in this pass.

Longer-term, deferred until asked for:
- A **Charm Upgrade** to pair with the new Charm Reforge — gear has
  both Upgrade (level a piece's rolled values) and Reforge; charms
  only have Reforge + Salvage so far. Revisit if charm power creep
  becomes an issue.
- A champion build-choice system (talent variants, stat-path choices,
  etc.) — nothing like this exists yet; a champion-facing Rework mode
  would need it first.
