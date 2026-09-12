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
| `machineborn_auto_mode_v1` | `{ encounterId: boolean }` — whether Auto Battle is armed for that mode; stays on until explicitly toggled off, survives Retry/Next Stage/resume | Persistent |

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
     'fang', 'squall', 'vex']` (2 Common + 2 Rare - Vex added as a 4th
     starter specifically so Chapter 1 can field a full 4v4 squad from
     the very first battle, once Campaign became 4v4 uniformly - see
     "Campaign 4v4/3-wave restructure" below). The original six
     Legendaries now live entirely in the real Summon pool alongside
     Bastian/Morwen/Talon/Raze; Bastian and Vara are additionally handed
     out as guaranteed, deterministic Campaign Chapter 1 stage-clear
     rewards (`CAMPAIGN_STAGE_CHAMPION_REWARDS`) rather than only via RNG.
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
    writes (all 21: equipment/gear/charm inventories, every currency,
    champion progress/roster, level cap, stage progress, the mid-battle
    save, the auto-battle skill-priority config, the last-fielded-squad
    memory, the per-mode Auto Battle on/off state) in one list, referenced
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
    - **Persisted PER MODE** (`autoModeByEncounter`,
      `machineborn_auto_mode_v1`): originally `autoBattleOn` reset to
      off on every `newBattle`/`resumeBattle` unconditionally, matching
      Raid's own Auto toggle - the user later asked for the opposite
      ("if auto abilities is toggled on it should be permanently on in
      that game mode until turned off again"), since re-toggling after
      every single Retry/Next Stage was pure friction. `autoModeByEncounter[encounterId]`
      is now the one persisted source of truth per mode: the team-select
      toggle (`#teamSelectAutoBtn`/`pendingAutoStart`) and the in-battle
      toggle (`#autoToggleBtn`) both write through to it immediately on
      click. `newBattle`'s `startInAuto` argument, when omitted (Retry/
      Next Stage don't pass one), falls back to `autoModeByEncounter[encounterId]`
      instead of forcing `false`; `resumeBattle` reads the same map for
      its encounter id. `autoBasicOnly` stays session-only/always-off-
      per-battle by design - it's a "just get through this specific wave"
      override, not a standing mode preference. `AUTO_MODE_KEY` is
      included in `allSaveKeys()` for New Game reset.
    - Manual skill/target clicks are ignored (`if (autoBattleOn)
      return;` guards on the `#skillRow`/`#enemyList`/`#teamList`
      click handlers) while Auto is on, rather than letting both
      inputs race — toggle Auto off to act manually again, matching
      Raid's own behavior.
    - **Team-select AUTO toggle** (`#teamSelectAutoBtn`): lets a fight
      start already in Auto mode instead of always having to toggle it
      on after Begin Battle. Opening team-select initializes
      `pendingAutoStart` from `autoModeByEncounter[encounterId]` (so it
      reflects that mode's persisted state, not always `false`), and
      flows through `beginEncounter`/`newBattle` as a `startInAuto`
      argument.
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
    named boss in the game — no parallel mechanic invented.
    - **First retune** (per explicit follow-up feedback): the initial
      pass (900 hp/40 atk/30 def, 3v3, single-target) was still
      clearable by a fresh Level-1 unlock squad in most attempts - too
      easy, missing the whole point of "not defeating him in 1 shot."
      Retuned to `hp: 2600, atk: 80, def: 35` and probed at 0/8 full
      clears, ~27.5% average depletion.
    - **Second retune — 5v5 and a per-turn Attack ramp** (further
      explicit feedback: "Boss should be 5v5 with ramping damage
      mechanics... needs a full team to deal with... early levels his
      stats are low enough your base team can do some minimal damage
      and get base rewards but you'll have to build a real team to beat
      him"). Main Boss's squad went from 3v3 to 5v5
      (`ENCOUNTERS.mainboss.teamIds`, 5 members) and the boss's basic
      attack from single-target to AoE (`aoe: true`, coeff dropped from
      0.85 to 0.5 to compensate for hitting all 5 instead of 1) so the
      *whole* squad has to survive him, not just whoever gets focused.
      The real difficulty lever is now `guildBossRamp(boss)`: an
      **unconditional** +`GUILD_BOSS_RAMP_STEP` (6%) Attack Up every
      Guild Boss own-turn, compounding, called from `takeTurn` right
      alongside `hollowKingMechanic` but gated on
      `currentEncounterId === 'mainboss'` so it never touches any other
      boss. Deliberately separate from `hollowKingMechanic` - Hollow
      King's Fury only ramps when it has a debuff to devour (and applies
      to every `isBoss` unit in the game); this one always fires,
      purely punishing a fight that runs long, regardless of debuffs.
      Stats retuned again for the bigger squad + AoE + ramp combination
      to `hp: 4400, atk: 62, def: 35`. Probed via `probe_guildboss.js`
      (8 trials, fresh Level-1 5-champion squad on Auto Battle, roster
      seeded with Bastian since a fresh save's 4 starters alone can't
      field 5): 0/8 full clears, ~34% average depletion (range 22-67% -
      real variance from the ramp mechanic, not just a flat number).
      `probe_guildboss_leveled.js` confirms the progression curve: a
      Level-25 squad reaches 96% depletion but still loses (the ramp
      genuinely threatens even a strong-but-not-strong-enough team, not
      just a weak one), while a Level-35 squad fully clears it -
      "beatable once you've actually built a real team," not "beatable
      just by grinding attempts." Still a first pass - expect iteration
      at other levels/gear tiers, same as every other hand-tuned
      encounter in this file. Main Boss now unlocks (Campaign Stage 4)
      one stage before the roster can actually field a full 5v5 (Bastian
      joins at Stage 6) - left as-is rather than moving the unlock,
      matching the existing "gate is visible, not mysterious" convention
      (team-select just shows 4/5 selected, Begin disabled, until
      Bastian joins two stages later).
19. **Auto-Level** (`autoLevelSquad`/`anyAutoLevelAvailable`, victory-
    screen `#autoLevelBtn`): the user asked for a way to auto-spend
    banked progression after a battle instead of clicking into the
    Armory every time. Repeatedly calls the Armory's own
    `levelUpChampion`/`ascendChampion` (never a parallel spend path) on
    every owned member of `currentTeamIds` until nothing more is
    affordable for any of them - reuses the exact same cost/cap checks
    those functions already enforce. Shown whenever `anyAutoLevelAvailable()`
    is true (a same-shaped, non-mutating check using the same
    affordability logic) regardless of win/loss - unlike Auto-Equip,
    this isn't tied to a drop or a victory, since leveling just spends
    whatever's already banked.
20. **Armory shows only owned champions**: both the gear/charm equip
    list (`armoryGrid`/`renderArmory`) and the Champions leveling/
    ascension list (`championList`/`renderChampions`) now filter
    `CHAMPIONS` through `isChampionOwned` before rendering, rather than
    showing every champion in the game with a disabled "Locked, summon
    to unlock" placeholder row for the ones you don't have yet.
    `championRowHtml`'s old locked-row branch was removed outright (dead
    once the list is pre-filtered) rather than left as unreachable code.
    Team-select's own champion grid is unaffected and still deliberately
    shows locked cards (so you can see what summoning would add) - this
    only applies to the two Armory panels.
21. **First-Clear Bonus** (`FIRST_CLEAR_BONUS_MULT`, `isFirstClear` in
    `endBattle`): the user's framing - Chapter 1's Stage 9 wall was
    softened (see the Campaign bullet below), but the deeper ask was to
    smooth the whole "push one mode to unlock/afford the next" cycle
    more generally, by rewarding *reaching* new content, not only
    grinding it: "give 1st time rewards from each stage of all the main
    content so that we progress slightly faster but also a bit more
    smoothly... push 1 content in order to get rewards to increase
    power to get through the next content's plateau." The exact first
    time any stage/level in any mode is cleared - `currentStage >
    prevCleared`, the identical `stageProgress`-driven check the unlock
    messaging already used - doubles every generic
    `ENCOUNTER_*_REWARD` payout for that battle (`rewardMult =
    stageRewardMultiplier(stage) * 2`) and guarantees any gear/charm
    drop that mode can ever produce. Never repeats on a replay of an
    already-cleared stage - a stage's normal (non-doubled, RNG-gated)
    reward is untouched on every subsequent clear, so this only
    accelerates *reaching* new content, not farming it.
    - **Reward purity preserved**: the guaranteed-drop bypass only
      applies when that mode's own `ENCOUNTER_DROPS` chance for that
      material is already > 0 - `(dropCfg.gearChance > 0 &&
      isFirstClear) || Math.random() < dropCfg.gearChance` and the
      mirror check for `charmChance`. An earlier version bypassed the
      RNG unconditionally and briefly made Tower's first clear drop a
      Charm (a material Tower never owns) - caught via
      `test_first_clear_bonus.js` and fixed; Tower still never drops
      charms, Faction Wars still never drops gear, even on a first clear.
    - **Main Boss is excluded from the bonus tag/multiplier entirely** -
      its Gold/XP/Shard/Summon Shard identity already lives entirely on
      the Guild Boss chest mechanic (`grantGuildBossChest`), not the
      generic `ENCOUNTER_*_REWARD` tables this multiplier touches, so
      showing "First-Clear Bonus! (2x rewards)" there would be pure
      noise - nothing would actually double. `stageProgress`/
      `isFirstClear` are still computed and drive its "Guild Boss Level
      N cleared" messaging exactly as before; only the reward-doubling
      tag is suppressed (`currentEncounterId !== 'mainboss'`).
    - `stageProgress[currentEncounterId]` is now updated at the *top* of
      `endBattle`'s `won` branch (it used to happen near the end, purely
      for the "stage cleared" messaging) so every reward calculation in
      between can read `isFirstClear` - a reordering, not a new
      persistence mechanism.
22. **Campaign Chapter 1 gear bootstrap**: a further explicit ask after
    the First-Clear Bonus above - "campaign chapter 1 grants 1x random
    gear piece on each 1st clear to speed up the process of getting
    gear on your initial champs." Campaign has no `ENCOUNTER_DROPS`
    entry at all (reward purity - it's Scrap+Gold-only, see Content
    framework below), so the generic First-Clear Bonus's guaranteed-
    drop bypass never touches it. Rather than giving Campaign a real
    `gearChance` (which would guarantee gear on every future chapter's
    first clear too, contradicting its documented "no farmable
    specialty" role), `endBattle` has a small Chapter-1-only carve-out:
    `if (currentEncounterId === 'campaign' && isFirstClear &&
    currentStage <= 12)` calls the exact same `rollGearDrop(currentStage)`
    Tower's own drops use (which itself calls `farmGear` under the
    hood) - so the granted piece is a real rolled instance (random
    slot/rarity/main stat/substats/set), landing in `gearInventory` and
    immediately eligible for Upgrade/Reforge/Rework/Salvage like
    anything farmed, not a fixed freebie with its own parallel
    machinery. Also sets `lastGearDrop` so the victory-screen Auto-Equip
    button offers it, same as any other gear drop. Scoped strictly to
    `currentStage <= 12` (Chapter 1) - Chapters 2-11 and beyond keep
    Campaign's existing no-gear reward purity untouched. Verified via
    `test_ch1_gear_and_unequip.js`: Stage 1's first clear grants exactly
    one rolled instance (confirmed via `mainStat`/`set`/`substats`
    fields), a replay of Stage 1 grants none, and a forced-win at Stage
    13 (Chapter 2) grants none.
23. **Direct gear Unequip**: the user asked to be able to "remove gear
    and swap onto another champ." This was already *possible* via
    `cycleGearSlot` (clicking a champion's own Armory slot button cycles
    forward through `[null, ...unclaimed instances]`, so cycling all the
    way around eventually reaches `null` and frees the piece) but
    clunky - freeing a specific piece meant clicking through however
    many other unclaimed items shared that slot first. The gear
    inventory list (`gearInventoryList`, alongside Upgrade/Reforge/
    Rework/Salvage) already showed which champion owned each equipped
    instance (`findGearOwner`) but had no direct way to detach it -
    Salvage was simply `disabled` while owned. Added `unequipGear
    (instanceId)`: walks every champion's `equipment` record and clears
    any slot pointing at that instance, in one click, from the same row
    that already displays the owner. A new "Unequip" button on each gear
    inventory row is enabled exactly when `findGearOwner` returns an
    owner (mirroring Salvage's existing disabled condition, inverted) -
    click it, then cycle the now-unclaimed instance onto a different
    champion's Armory row same as before. `cycleGearSlot` itself is
    unchanged - this adds a direct detach action alongside it rather
    than replacing the cycle-to-equip mechanism. Verified via
    `test_ch1_gear_and_unequip.js`: farm a piece, equip it on Zephyr,
    confirm Unequip is enabled and clicking it clears Zephyr's slot,
    then confirm the freed piece can be cycled onto Fang.

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
| **Campaign** (`campaign`) | 4v4 × 3 waves, every chapter including Chapter 1 | 132 fixed hand-authored stages (`CAMPAIGN_CHAPTERS` = `CAMPAIGN_CH1_STAGES` (12) + 10 chapters × 12 stages each via the `buildChapterArc` generator — see "New-game onboarding" and "Campaign 4v4/3-wave restructure" below) — **not** the infinite ladder, no stat compounding, replaying an old stage is always the same fight | Always open | Scrap + Gold only (bootstrap), plus 1 rolled gear piece on each of Chapter 1's 12 first clears only (see Systems item 22) |
| **Tower** (`tower`) | 5v5 | Infinite stage ladder | Stage 7 | Gear (guaranteed) + Gear Rework Catalysts (rare, rides gear drops) + Scrap |
| **Faction Wars** (`faction`) | 4v4 × 3 waves | Infinite stage ladder | Stage 10 | Charms (rolled chance) + Charm Dust (guaranteed) |
| **Main Boss** (`mainboss`) | 5v5 vs. 1 Guild Boss (AoE basic, Attack ramps every own turn) | "Levels" (reuses the infinite-ladder `stageProgress` mechanism — a level IS a stage) | Stage 4 | Gold + XP + Shards + Summon Shards, paid out via 4 chests unlocked at 75%/50%/25%/0% of the boss's health, ramping in value per chest and per level (see "Main Boss: Guild Boss redesign" below) |
| **Dungeon** (`dungeon`) | 4v4 | Infinite stage ladder | Stage 132 (Campaign complete) | Ascension Cores only |
| **Galactic War** (`galactic`) | 5v5 × 5 waves | Infinite stage ladder, no retreat between waves within one attempt | Stage 132 (same threshold as Dungeon — a "you finished Campaign" bonus node, not a fifth step in the unlock sequence) | Bulk Scrap + Gold + XP only — no gear/charms/shards/catalysts/Ascension Cores |
| **Grand Arena** (`grandarena`) | 3v3 vs. one of 4 AI archetype squads | Infinite stage ladder, stage number cycles through the 4 archetypes (`(stage - 1) % 4`) forever | Stage 132 (alongside Dungeon/Galactic War) | Arena Medals only — spent on Arena Pulls (Vanguard-set gear), never dropped directly |

`CONTENT_UNLOCKS = { mainboss: 4, tower: 7, faction: 10, dungeon:
CAMPAIGN_CHAPTERS.length, galactic: CAMPAIGN_CHAPTERS.length,
grandarena: CAMPAIGN_CHAPTERS.length }` — keyed to Campaign's flat
stage index now, not a "chapter number" (Chapter 1 alone spans 12
stages). Main Boss deliberately unlocks *before* Tower despite Tower's
old head-start — originally because Main Boss's squad (3v3 at the
time) matched the 3-champion starter roster exactly while Tower's 5v5
didn't; Main Boss has since become 5v5 itself (see Systems item 18's
"Second retune"), so it now unlocks one stage before it's actually
fully fieldable (Stage 6's Bastian is the roster's 5th member) — see
"New-game onboarding" and that Systems item for both the original
reasoning and the current state.

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

**Superseded-details note**: this section documents Chapter 1 as it
existed BEFORE the Campaign 4v4/3-wave restructure (see that section
below, which now applies to Chapter 1 too, not just Chapters 2-11) -
"3-champion starter roster," "3v3," and Stage 9 being a single 3-enemy
fight all describe that earlier shape. The roster is now 4 starters
(`STARTER_CHAMPION_IDS` includes Vex), every stage is 4v4/3-wave
(single-wave only at Stage 12), and Stage 6 now has a real mini-boss
(Cave Warden) rather than being a plain stage that happens to also
grant Bastian. The stage-clear rewards, unlock sequencing, and overall
narrative arc described below are all still accurate and still fire at
the same stage numbers - only the battle format underneath changed.
Separately, "Stage 9 was deliberately built as a wall the squad cannot
reliably clear" (below) is ALSO now stale, on top of the format change
- see "Chapter 1 Stages 9-12 retune" further down: that intentional
wall was later removed entirely in favor of a smooth ramp, per direct
follow-up feedback.
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
- **Chapter 1 was initially excluded, then included in a direct
  follow-up.** The first pass kept Chapter 1 3v3/single-wave, unchanged,
  reasoning that requiring 4 champions before Stage 6 (the stage that
  used to grant the roster's 4th member, Bastian) would brick new-game
  onboarding for a fresh 3-starter save. The user's next message
  rejected that workaround directly: "simply give a 2nd rare champ at
  the beginning to have the 4th champ." `STARTER_CHAMPION_IDS` gained
  Vex (already an existing Rare champion, not a new one) as a 4th
  starter, so Chapter 1 can field 4v4 from the very first battle like
  every other chapter - `squadSizeFor('campaign', stage)` now simply
  returns 4 unconditionally, no Chapter-1 special case. Chapter 1's own
  12 stages were rewritten with `chapterOneWaveStage(idPrefix, regular,
  special)` - the same "regular ramp / mini-boss at 6 / boss alone at
  12" shape `buildChapterArc` uses for Chapters 2-11, but built directly
  from Chapter 1's own already-existing per-stage enemies (there was no
  separate A/B/C trio to feed a generator from, since Chapter 1 was
  always 12 distinct hand-authored stages, not 3). Stage 6 gained a
  genuinely new mini-boss, "Cave Warden" (never existed before this
  pass), joining 3 Cave Lurkers in the finale wave - Bastian's grant on
  clearing Stage 6 is unaffected (`CAMPAIGN_STAGE_CHAMPION_REWARDS`
  fires on stage number, independent of enemy composition). Stage 9's
  "wall" concept and Stage 12's capstone both carry over structurally,
  just reformatted (Stage 12 gained a 4th Warcamp Guard to fill 4v4).
  **Difficulty required a second pass**: converting each stage straight
  into 3 full-strength waves of 4 (no HP refill between waves) roughly
  quadrupled total attrition versus the original single 3-enemy fight,
  turning even a boosted 6-champion roster into a reliable 0/8 at Stage
  9-10 (probed and confirmed too hard, well past the intended
  difficulty). Fixed by discounting `chapterOneWaveStage`'s two warm-up
  waves to 55% of the stage's own stats, leaving the finale wave at full
  strength - the same "the real fight is the last wave" shape the
  mini-boss/boss stages already use. Re-probed via
  `probe_ch1_wavestages.js`: Stage 10 recovered to 3/4 wins (a real but
  passable relief step, matching its original design intent), Stage 9
  stayed a deliberate 0/4 wall (matches its documented intent - a
  6-champion-but-still-Level-1 roster isn't supposed to clear it; actual
  leveling/gearing via Main Boss/Tower is the intended unlock). Not an
  exhaustive per-stage sweep of all 12 stages - same "first pass, expect
  iteration" caveat as everything else newly restructured in this file.
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
  a stage entry is either a flat template array (one wave - now only
  true for every chapter's Stage 12, boss-alone stages) or
  `{ waves: [...] }` (Stages 1-11 of every chapter, Chapter 1 included)
  - resolved generically, no per-mode branching needed since the
  underlying wave-advance loop in `checkBattleEnd` was already generic
  (the same mechanism Faction Wars/Tower/Galactic War already use for
  their own multi-wave configs).
- **`CAMPAIGN_CHAPTER_LENGTHS`** is now `[12, 12, 12, ..., 12]` (Chapter
  1's 12 plus 12 for each of the 10 chapters after it) instead of
  `[12, 3, 3, ..., 3]` - `campaignChapterInfo`/`campaignStageLabel`/
  `campaignClearMessage` needed zero code changes, since they already
  read chapter boundaries generically off this array. Total Campaign
  length: 12 + 10×12 = **132 stages** (up from 42).
  `CAMPAIGN_CHAPTERS.length` still drives `CONTENT_UNLOCKS.dungeon/
  galactic/grandarena` automatically - they now unlock at Stage 132.
- **Verified via `test_five_features.js`** (Chapters 2-11 shape, before
  Chapter 1 was also converted) and `test_round2_features.js` (Chapter
  1 specifically, both scratchpad, not committed): confirms
  `CAMPAIGN_CHAPTER_LENGTHS`/total stage count, that Campaign is 4v4 at
  every stage including Chapter 1, that a Chapter 2 regular stage is a
  3-wave/4-enemy fight, that Chapter 2's boss stage (global Stage 24) is
  a single wave with Grunt Warchief present and `isBoss:true`, that
  Chapter 2's mini-boss stage (global Stage 18) is 3 waves with Grunt
  Brute in the finale wave without `isBoss`, and the equivalent checks
  for Chapter 1's own Stage 1/Stage 6 (Cave Warden)/Stage 12 (4 enemies,
  Chapter Warlord `isBoss:true`). Not an exhaustive sweep of every
  stage, matching this file's established testing philosophy for large
  generated/restructured content batches.

**Chapter 1 Stages 9-12 retune (removing the deliberate wall).** A
later explicit follow-up: "Campaign is a bit too strong in ch1. We
want 9-12 to be slightly above 8." Stage 9 had been deliberately built
(see "New-game onboarding" above) as a hard wall a fresh squad could
not reliably clear - `Warband Enforcer` at 190 hp/64 atk/40 def, more
than double Stage 8's numbers, then Stage 10 written as a *relief*
step below it. That "wall → relief → wall → capstone" shape was the
original design intent, but the user later judged it too harsh once
layered on top of the already-tougher 4v4/3-wave format. Replaced with
a smooth incremental ramp - each of Stages 9-11 a modest step above
the last (Stage 9's `Warband Enforcer` down to 100 hp/40 atk/28 def,
Stage 10/11 stepping up gently from there), Stage 12's boss still a
real capstone bump over its own escorts but proportionate (220 hp vs.
their 130, not the previous 280 vs. 120) rather than a second brick.
Probed via `probe_ch1_9to12_v2.js` (scratchpad, 4 trials each, the
actual roster available by Stage 9+ in real play - 4 starters +
Bastian + Vara, all Level 1, on Auto Battle): 4/4 at every one of
Stages 9-12, confirming they're genuinely passable now rather than a
deliberate brick. There is no longer a single designed-to-be-uncleared
stage anywhere in Chapter 1 - the push-and-return loop for new players
is now carried more by the First-Clear Bonus above (which pays out
right as each new stage is *reached*, not gated behind an intentional
wall) than by one stage nobody can beat on arrival. Not an exhaustive
sweep - only Stages 9-12 were re-probed, matching this file's "first
pass, expect iteration" convention for hand-tuned content.

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
  that actually own that material. **Stale as of the Guild Boss
  redesign** (Systems item 18): `ENCOUNTER_GOLD_REWARD`/`_XP_REWARD`/
  `_SHARD_REWARD`/`_SUMMON_SHARD_REWARD` no longer have a `mainboss` key
  at all - that whole reward identity moved onto the chest mechanic
  (`grantGuildBossChest`), so a generic-table entry there would
  double-pay. `ENCOUNTER_GOLD_REWARD` is now just `{ campaign: 10,
  galactic: 90 }`; `ENCOUNTER_XP_REWARD` just `{ galactic: 70 }`;
  `_SHARD_REWARD`/`_SUMMON_SHARD_REWARD` are both empty objects.
  Galactic War remains the one intentional multi-mode overlap (bulk
  Scrap+Gold+XP, no specialist material of its own) - never a
  specialist material like gear/charms/shards/Ascension Cores/Arena
  Medals. `ENCOUNTER_ARENA_MEDAL_REWARD` has only `grandarena`, and
  there is no `ENCOUNTER_DROPS.grandarena` entry at all — Grand Arena's
  gear payoff (`pullArenaGear`) is a manual Armory spend, never a
  victory-screen roll, so the reward line itself stays Medals-only.
  `endBattle` guards every reward block with `if (reward > 0)` so a
  mode that doesn't grant something never shows a useless "+0 X" line.
  The First-Clear Bonus (Systems item 21) respects this same purity
  rule for its guaranteed-drop bypass - see that item for how.
  Campaign's own Chapter 1 gear bootstrap (Systems item 22) is the one
  deliberate, narrowly-scoped exception to Campaign's own "no gear"
  purity - a special-cased `currentStage <= 12` check in `endBattle`,
  not an `ENCOUNTER_DROPS.campaign` entry (which would have applied to
  all 132 stages, not just Chapter 1's 12).
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
- Chapter 1 itself was initially kept OUT of that restructure and left
  3v3/single-wave — requiring 4 champions before Chapter 1 Stage 6
  (the stage that granted Bastian, the roster's 4th member at the time)
  would have bricked new-game onboarding for a fresh 3-starter save.
  This was a judgment call made during implementation, not something
  the user was asked about directly. The user's next message rejected
  the workaround directly and asked for Chapter 1 included too, via the
  simplest fix: a 2nd Rare starter (Vex) so the roster is 4 from the
  very first battle - see the follow-up round below and Systems item 1
  (`STARTER_CHAMPION_IDS`).

**Second follow-up round — built.** Four more asks plus a correction to
the item above, addressed in the same session:
1. **Auto Battle should stay on per mode until manually turned off** -
   previously reset to off every single battle. See Systems #15's
   "Persisted PER MODE" bullet (`autoModeByEncounter`,
   `machineborn_auto_mode_v1`).
2. **Armory should only show owned champions** - previously showed
   every champion with a disabled "Locked" row for unowned ones. See
   Systems item 20.
3. **The Guild Boss was still too easy** - a fresh Level-1 unlock could
   often fully clear it, undermining the whole "chest per quarter,
   real progress needs external growth" point. Retuned (900→2600 hp,
   40→80 atk, 30→35 def) and reprobed at 0/8 full clears, ~27.5% average
   depletion - see Systems item 18's retuning bullet.
4. **An auto-leveling/promoting button after battle** - see Systems
   item 19 (`autoLevelSquad`).
5. **The Chapter 1 exclusion from item 5 above was reversed** - "simply
   give a 2nd rare champ at the beginning to have the 4th champ." Vex
   joined `STARTER_CHAMPION_IDS`, Chapter 1 was rewritten to 4v4/3-wave
   with a new Stage 6 mini-boss (Cave Warden), and the resulting
   difficulty spike from tripling wave count was corrected with a
   warm-up-wave stat discount - see the Campaign 4v4/3-wave restructure
   section's own follow-up bullet under "Content framework" above for
   the full mechanism and probe numbers.

**Third follow-up round — built.** One more explicit correction to the
Guild Boss, since item 3 above still wasn't landing the intended feel:
"Boss should be 5v5 with ramping damage mechanics. Goal is to balance
him so he's hard and needs a full team to deal with. Early levels his
stats are low enough your base team can do some minimal damage and get
base rewards but you'll have to build a real team to beat him." Main
Boss went from 3v3 to 5v5, the boss's basic attack from single-target
to AoE, and a new unconditional per-own-turn Attack ramp
(`guildBossRamp`, +6%/turn compounding) was added as the actual
difficulty engine - see Systems item 18's "Second retune" bullet for
the full mechanism, stat numbers (4400 hp/62 atk/35 def), and probe
results (0/8 full clears at Level 1, ~34% average depletion; a
Level-25 squad reaches 96% but still loses; a Level-35 squad clears).
The user also restated the core design mandate directly: *"this game
needs to be mechanically driven and balanced enough that it's a
constant plateau → divert to another game mode to push to get stronger
via rewards then cycle through all content this way."* Keep this as
the standing bar for every future difficulty pass, not just Main
Boss's - a mode is tuned correctly when it stops a fresh/underpowered
squad cold while staying genuinely clearable by a squad that actually
detoured through the rest of the content loop to grow first.

**Fourth follow-up round — built.** Two related asks, both about the
push-and-return loop feeling too harsh/choppy specifically in Chapter
1: "Campaign is a bit too strong in ch1. We want 9-12 to be slightly
above 8... we should maybe give 1st time rewards from each stage of
all the main content so that we progress slightly faster but also a
bit more smoothly... push 1 content in order to get rewards to
increase power to get through the next content's plateau." Both built:
1. **Chapter 1 Stages 9-12 retuned from a deliberate wall into a smooth
   ramp** - see "Chapter 1 Stages 9-12 retune" under "Content
   framework" above for the numbers and probe results (4/4 at every
   stage 9-12 now, vs. the previous intentional 0/4 wall at Stage 9).
2. **First-Clear Bonus** - a one-time doubled reward + guaranteed drop
   the exact first time any stage/level in any mode is cleared, across
   the whole game, not just Campaign - see Systems item 21 for the full
   mechanism, the reward-purity bug caught and fixed during testing
   (an unconditional guarantee briefly let Tower drop charms, a
   material it never owns), and why Main Boss is deliberately excluded
   from the bonus tag (its rewards already live entirely on the Guild
   Boss chest mechanic, untouched by this multiplier).

**Fifth follow-up round — built.** Three more asks, continuing the same
"speed up the early gear/power loop" thread as the Fourth round above:
"I would suggest campaign chapter 1 grants 1x random gear piece on each
1st clear to speed up the process of getting gear on your initial
champs. Also we should be able to remove gear and swap onto another
champ. Also we need the roll/upgrade progression from gear." All three
addressed:
1. **Campaign Chapter 1 gear bootstrap** - Chapter 1's 12 first-clears
   each now guarantee one gear piece, via the same `rollGearDrop`/
   `farmGear` path Tower's own drops use - see Systems item 22.
2. **Direct gear Unequip** - a one-click "Unequip" button on the gear
   inventory list detaches a piece from whichever champion has it
   equipped, so it can be cycled onto a different champion without
   hunting through `cycleGearSlot`'s cycle order to free it first - see
   Systems item 23.
3. **"Roll/upgrade progression from gear"** - read as: the Chapter 1
   grant above needed to be a real rolled instance that feeds the
   existing Scrap Upgrade/Reforge/Rework economy (levels 0-12, substat
   rerolls, etc. - see Systems items 6/8), not a fixed freebie outside
   that system. Satisfied for free by reusing `rollGearDrop`/`farmGear`
   for item 1 above rather than building a separate reward path - the
   granted piece has a real main stat/substats/set roll and a `level`
   field from the moment it's granted, so it immediately shows up in
   the gear inventory list with working Upgrade/Reforge/Unequip/Salvage
   buttons like anything farmed. No separate "upgrade progression"
   system was needed since the existing one already covers it once the
   grant is a genuine instance.

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
- `GUILD_BOSS_WAVE`'s stats (4400 hp/62 atk/35 def, 5v5/AoE/ramp,
  retuned twice already per explicit feedback), `GUILD_BOSS_RAMP_STEP`
  (6%/turn), and `guildBossChestReward`'s payout curve are still only
  probed at Level 1 vs. Level-1/25/35 squads — expect retuning at the
  levels in between and at higher gear tiers once real play exercises
  them.
- Several pre-existing scratchpad Playwright tests (not committed —
  see "Testing methodology") hardcode the old 42-stage Campaign total
  for unlock-gating seeds (`galactic_war`, `grand_arena`,
  `roster_tier`, `dungeon_ascension`, `ch1_rework`,
  `campaign_expand_basic`, and — newly confirmed while regression-
  testing the Chapter 1 gear bootstrap/Unequip round below,
  `TOTAL_STAGES = 42` still hardcoded — `test_content_framework.js`)
  and are now stale, not failing due to a real regression — same
  "stale test, not a regression" convention as always, just not all
  individually fixed in this pass. (Several other
  scratchpad tests that manually built a 3-champion Campaign squad -
  `test_factions.js`, `test_auto_priority.js`, `test_new_champions.js`,
  `test_content_framework.js`, `test_galactic_war.js` - WERE fixed this
  round, since Campaign's squad size itself changed to 4 everywhere.)
- Chapter 1's difficulty curve is a first pass through two retunes now
  (the warm-up-wave discount, then the Stages 9-12 wall removal) - only
  Stages 9-12 were re-probed after the second retune (now 4/4 at each,
  see "Chapter 1 Stages 9-12 retune"); Stages 1-8 were not individually
  re-verified since the warm-up-wave discount pass.
- The First-Clear Bonus's 2x multiplier and guaranteed-drop are a first
  pass, untested at scale (i.e. whether doubling every mode's very
  first Tower/Faction Wars/Dungeon/Galactic War/Grand Arena clear meshes
  well with the roster-tier/stage-cap pacing elsewhere) - revisit the
  multiplier if pushing through early game ends up feeling too fast
  once real play exercises the whole loop.

Longer-term, deferred until asked for:
- A **Charm Upgrade** to pair with the new Charm Reforge — gear has
  both Upgrade (level a piece's rolled values) and Reforge; charms
  only have Reforge + Salvage so far. Revisit if charm power creep
  becomes an issue.
- A champion build-choice system (talent variants, stat-path choices,
  etc.) — nothing like this exists yet; a champion-facing Rework mode
  would need it first.
