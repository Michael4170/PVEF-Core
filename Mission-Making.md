# Building a PVEF mission

How to stand up a new PvE mission on PVEF Core. Written against the live project (`PVEF Core`, GUID `6A22BCD5749BFA2B`) and the shipped reference world `PVEF Arland`, so every path, GUID and attribute name below is one that currently exists — not one the plan said it would.

**Read this once end to end before you place anything.** Roughly half the steps exist because of a failure that is silent — the mission boots, looks right, and does the wrong thing hours later.

---

## 0. What you are actually building

A PVEF mission is **four things**, and only the first is unusual:

| | What | Where it lives |
|---|---|---|
| 1 | A **world** — a sub-scene, with bases, garrisons and managers | `Worlds/<Name>.ent` in your addon |
| 2 | A **game mode entity** carrying `PVEF_Manager` | placed in the world's Managers layer |
| 3 | A **mission header** `.conf` | `Missions/<Name>.conf` |
| 4 | A **dependency on PVEF Core** | your `addon.gproj` |

There is no terrain-profile config file to write. Everything PVEF needs from the map it either reads off the bases at boot or derives from their spacing. What you configure, you configure on `PVEF_Manager` in the World Editor.

### Which world to sub-scene from

| Path | Parent | You author | Use when |
|---|---|---|---|
| **A** | vanilla Conflict mission world (`worlds/MP/Campaign_Arland.ent`) | nothing — BI's bases are already placed | fastest start, but BI reworks those layouts and your overrides shift under you |
| **B** | the bare **terrain** world (`worlds/Arland/Arland.ent`) | every base, by hand | modded terrains — it is the only option there |

**`PVEF Arland` is path B.** Its world file is two lines:

```
SubScene {
 Parent "{A9806AF617972E97}worlds/Arland/Arland.ent"
}
```

Path B is ~20 base placements of work. The tier prefabs in §4 are what make that a day rather than a week.

---

## 1. Addon setup

New addon, one dependency: PVEF Core. Your `addon.gproj` should read like PVEF Core's own, with PVEF Core's GUID added:

```
GameProject {
 ID "MyMissionAddon"
 GUID "<your new guid>"
 TITLE "My Mission"
 Dependencies {
  "58D0FB3206B6F859"          // base game
  "6A22BCD5749BFA2B"          // PVEF Core
 }
}
```

If your terrain is a modded map, its addon GUID goes in the list too.

**Put the addon under version control before you place the first base.** Twenty bases of world edits with no history is a bad place to discover a mistake.

---

## 2. The world, and the five entities that must be in it

Create your world as a sub-scene of the terrain world, then make a **`Managers` layer** and put these five in it. This list is copied from `PVEF Arland_Layers/Managers.layer` — it is the complete set, and four of the five fail *silently* if missing.

| Entity | Prefab | Missing = |
|---|---|---|
| `SCR_AIWorld` | `{E0A05C76552E7F58}Prefabs/AI/SCR_AIWorld.et` | AI cannot path anywhere. Looks like broken AI, not missing config. |
| `PerceptionManager` | `{028DAEAD63E056BE}Prefabs/World/Game/PerceptionManager.et` | AI cannot see. |
| **Game mode** | `{ABE4F8FEE9FCD1E2}Prefabs/MP/Modes/Conflict/GameMode_PVEF_Conflict.et` | no framework at all |
| `RadioManagerEntity` | `{B8E09FAB91C4ECCD}Prefabs/Systems/Radio/RadioManager.et` | radio coverage never works; base connection lines never draw |
| `SCR_CampaignFactionManager` | `{F1AC26310BAE3788}Prefabs/MP/Campaign/CampaignFactionManager.et` | no factions |
| `SCR_LoadoutManager` | `{58FBD035E53D95C1}Prefabs/MP/Campaign/CampaignLoadoutManager.et` | no loadouts |

`GameMode_PVEF_Conflict.et` already inherits vanilla `GameMode_Campaign.et` and already carries `PVEF_Manager`. **Do not add `PVEF_Manager` yourself** — one per world, and it is in the prefab.

On the placed instance you set the victory threshold. PVEF Arland uses:

```
m_iControlPointsThreshold 10
```

That is "players hold 10 control points and the round ends". Count your capturable bases and pick a number below that — 10 of 10 on Arland means taking the island.

### Layers

Organise by **system**, not per base. PVEF Arland uses four:

```
Managers        the six entities above
Bases           every base, with its garrisons and counters parented under it
AI_Bases        loose AI content
StartingPoints  the player MOB and the offshore anchor
```

You can toggle a whole subsystem while editing, and related entities stay together.

---

## 3. Navmesh — and the shortcut that is legitimate

AI pathing needs `SCR_AIWorld`'s `NavmeshWorldComponent`s to *point at* `.nmn` files. Generating a navmesh is not enough; if nothing points at it the log says *"No navmesh file specified! Will initialize empty navmesh world"* and every AI stands still.

**If your terrain is unmodified vanilla, reuse BI's navmesh instead of generating one.** PVEF Arland does exactly this — three `NavmeshWorldComponent`s pointing at:

```
{D8EF7131FB31AF97}worlds/GameMaster/Navmeshes/GM_Arland.nmn          soldiers
{A0AEEB1E7EF474FA}worlds/GameMaster/Navmeshes/GM_Arland_vehicles.nmn vehicles
{804386FFEB7B7EFD}worlds/MP/Navmeshes/LowResArland.nmn               low-res
```

The Game Master navmeshes cover the whole terrain and BI maintains them. This works because path B changes no terrain — you place buildings *on* the existing heightmap.

**You must generate your own the moment you change the ground.** Flattening under a base, Terrain Tools → Bake Selection, or placing a composition that cuts into a slope all invalidate the baked mesh locally. Then: Navmesh Tool → Connect → Generate (or *Rebuild changed tiles*), and **tick "Autosave when done" or click Save afterwards** — without that the generated mesh is discarded and you have done nothing.

---

## 4. Bases

### The tier prefabs

PVEF Core ships five. Place these, not vanilla base prefabs — they already carry `PVEF_BaseTag` with the right tier and the right `SCR_CampaignMilitaryBaseComponent` flags.

| Prefab | GUID | Tier | Notes |
|---|---|---|---|
| `PVEF_Base_Large.et` | `{B84BC6CC1A1D23C3}` | LARGE | major military site |
| `PVEF_Base_Small.et` | `{E59C25F0ECF1F91F}` | SMALL | ordinary capture objective |
| `PVEF_Base_Relay.et` | `{91228F3BEE581346}` | RELAY | radio relay; ships a tent + table as dressing |
| `PVEF_Base_Mob.et` | `{EA543E8593AE68A9}` | *(inferred)* | player main operating base. One per map. |
| `PVEF_Base_USSR.et` | `{BE5ECEEC6A9FF616}` | *(inferred)* | the offshore OPFOR anchor. One per map. |

All live under `Prefabs/Systems/MilitaryBase/` in PVEF Core.

They are **logic only** — `PVEF_Base_Small.et` is six lines. Buildings, bunkers, sandbags and MG nests are separate compositions you place around each site to suit the terrain. Keep that separation: a tier prefab that bundles buildings makes every base on the island look identical and drags Everon's architecture onto maps where it makes no sense.

### On every capturable base, set exactly one thing

```
PVEF_BaseTag
  m_sProfileId    "Mosshill"          <-- REQUIRED. Unique in the world.
```

The tier is already correct from the prefab. `m_bCapturable`, `m_bStagingEligible` and `m_bCountsForVictory` all default to true and are right for an ordinary base.

**Why the profileId is mandatory and why it must not be the display name.** Conflict randomises in-game base names every session — the same base reads as CHICAGO one run and COMET the next. `m_sProfileId` is the stable key everything else attaches to. Get it wrong and per-base config silently attaches to the wrong base after a restart, which is the kind of bug that takes a week to see.

You will also want the display name, on the base component itself:

```
SCR_CampaignMilitaryBaseComponent
  m_sBaseName       "Mosshill"
  m_sBaseNameUpper  "MOSSHILL"
```

> **One habit worth adopting that PVEF Arland does not:** keep the entity name, the profileId and the display name identical. On Arland they have drifted — the entity `Arleville_Heights_Relay` carries profileId `Military_Base_Relay`, and `Main_Military_Relay` carries `Airfield_Relay`. Nothing is broken by this, but every log line names the profileId, and when you are reading a hundred of them at 1 Hz you want the name in the log to be the name in the hierarchy.

### Relay radio range

Relays are what make the front a front. Each PVEF Arland relay sets:

```
SCR_CoverageRadioComponent
  Transceivers > RelayTransceiver > "Transmitting Range"  1000
```

1000 m on a 4 km island. On a bigger map, raise the ranges or place more relays — and see `m_fRadioRangeScale` in §8 for the global multiplier that saves you editing every base.

### The two HQs — no tag needed

`PVEF_Base_Mob` and `PVEF_Base_USSR` carry **no `PVEF_BaseTag`**, deliberately. PVEF infers both from `IsHQ()` plus faction, and the inference forces the right values:

- **Player MOB** → not capturable, not staging-eligible, does not count for victory.
- **Offshore anchor** → the same, and critically **staging-eligible = false**. If the anchor could stage counter-attacks, reinforcements would spawn in open sea and swim. Inference closes that off; nothing you can mis-tick.

Both are placed in `StartingPoints`, both with `m_bCanBeHQ 1`.

### The offshore anchor — the step nobody guesses

**Place the OPFOR HQ in the water, off the playable area.** Not at the far end of the island. PVEF Arland's sits at `[-263, -97.9, 626]` — negative X, well below sea level, outside the map.

The reasoning, in order:

1. Conflict requires every capturable faction to have an HQ. It anchors radio range and the faction's campaign logic.
2. If OPFOR has bases but no HQ, **the engine establishes one at a random island location.** That is a real failure mode, not a theoretical one.
3. So OPFOR gets an HQ that is unreachable — and unreachable means uncapturable. Players can never end the campaign by seizing it.

**`m_bCanBeHQ` must be `false` on every other base you place.** That flag is the actual mechanism stopping the engine promoting one of your island bases to enemy HQ. The tier prefabs already set it correctly; if you build a base from scratch, this is the one to check.

Also note: the vanilla HQ composition ships four hand-placed soldiers. At the anchor they stand in the sea, spending AI budget from frame one, and they cannot be gated because there is no spawn point to eliminate. PVEF deletes them at boot (`m_bStripAnchorAI`, on by default) — the player MOB's own units are never touched.

---

## 5. Garrisons

**The whole rule is: place an ambient patrol spawn point at a base and it is gated.**

Use `AmbientPatrolSpawnpoint_USSR.et` (`{A73205DEA8361F26}`), OPFOR-affiliated from birth — one less runtime mutation than the FIA variant vanilla bases ship with.

Three ways a spawn point gets claimed by a base, in priority order:

| Source | Meaning |
|---|---|
| `remnant` | vanilla's own registered list. Authoritative where it exists. |
| `child` | parented under the base entity. **Explicit intent — no distance test, works at any range.** |
| `proximity` | anything else inside the derived claim reach, taken by the *nearest* base |

**Parent it.** That is the one remedy that always works and never asks you to reason about a reach formula. Proximity adoption is a convenience, not the contract.

### Two authoring rules that are not obvious

**Give a garrison one waypoint or none.** A spawn point with `m_iRespawnPeriod == 0` and a child `AIWaypointCycle` has a **50% chance of spawning its group at a random waypoint of the cycle** instead of at its post. If the route loops outside the compound, half your sessions have the defenders outside the compound. `PVEF_BaseValidator` reports the furthest waypoint distance for any such spawn point at startup — read that line.

**Never hand-place an `AIGroup`.** A group dragged straight into the world is always-on, sits outside the engine's proximity management and its AI budget, and is invisible to the gate. PVEF detects and names them at startup; it does not adopt them.

### Attributes for a base garrison

| Attribute | Value | Why |
|---|---|---|
| `m_eGroupType` | `FIRETEAM`, `SQUAD_RIFLE`, … | must exist in the OPFOR catalog — USSR ships FireGroup, MachineGunTeam, RifleSquad, SentryTeam, Team_AT |
| `m_iRespawnPeriod` | `0` | a cleared base stays cleared. Non-zero puts the base on a genuine respawn timer, which is a legitimate choice for ground you want re-contested. |
| `m_eImportance` | leave it | PVEF sets it (NORMAL for garrisons) and names anything it overrides |
| `m_iSpawnDistanceOverride` | `-1` | inherit vanilla's 600 m |
| `m_iDespawnDistanceOverride` | `-1` | inherit vanilla's 800 m |

A group type absent from the faction's catalog resolves to an empty prefab, marks the spawn point spawned, and fields **nothing, silently**. The startup catalog dump prints `groupType -> prefab` per entry so you can check what exists rather than guessing.

### Roaming patrols

A patrol that should stay always-on and never go dark with an objective gets `PVEF_RoamingPatrolTag` on the spawn point, plus `m_eImportance = LOW` so it yields budget to real fights. A patrol placed well away from every base needs no tag — nothing is in range to claim it.

**An unclaimed spawn point is not free.** It never gets gated, so it holds a slot in the ambient system's rotation all round — which slows every real garrison down *and* inflates the arming radius. Three orphans on a 13-garrison front costs about three seconds of extra wait per objective. Read the "unclaimed" lines at startup; they name coordinates, nearest base and shortfall in metres.

---

## 6. Counter-attacks

### Place a template. Do not build one.

Three ready-made spawn points under `Prefabs/Systems/AmbientPatrol/`:

| Prefab | Force | `m_iGroupMultiplier` | Group prefab |
|---|---|---|---|
| `AmbientPatrolSpawnpoint_USSR_12` (`{A73205DEA8361F27}`) | 12 | 3 | `PVEF_Group_USSR_Counter_12` |
| `AmbientPatrolSpawnpoint_USSR_24` | 24 | 6 | `PVEF_Group_USSR_Counter_24` |
| `AmbientPatrolSpawnpoint_USSR_48` | 48 | 12 | `PVEF_Group_USSR_Counter_48` |

Each already carries `PVEF_CounterAttackTag`, the matching group prefab, a budget figure that matches it, and a defend waypoint child **with a `Hierarchy` component**.

**Your whole job is three actions:**

1. Drag the template into the world.
2. **Parent it to the base it attacks.**
3. Move its child defend waypoint onto the ground you want held.

Place it where the attack should come *from* — the spawn position is the author's decision, not a random bearing. PVEF deliberately does not randomise it: you picked that spot by reading terrain, which beats five procedural tests.

### The one thing that will bite you

**A nested entity needs a `Hierarchy` component or it is never a runtime child.** This is the single fault that cost most of M2 — the waypoint existed, was correctly placed, and was invisible. The shipped templates have it. If you ever build a spawn point from scratch, this is the thing to get right, and the "no waypoint parented" warning names it explicitly.

### Tuning a placed counter-attack

| Attribute | Default | What it does |
|---|---|---|
| `m_iRespawns` | 2 | respawns **after** the first wave. `2` = three waves total. `-1` = endless. |
| `m_iRespawnSeconds` | 30 | gap after a wave is wiped before the next is released. `0` = framework default (15 s). |
| `m_fObjectiveRadius` | 100 | how much ground the force holds. **15 m makes twelve men stand on one spot.** |
| `m_iGroupMultiplier` | matches template | AI budget reserved per wave, in fours. **Does not resize the force.** |
| `m_sGroupPrefab` | matches template | what actually decides the size |
| `m_fPlayerClearRadius` | 0 → 212 m | how close a player must be to *hold* a wave. Not lost — it fires on a later pass. |
| `m_sTargetBaseId` | empty | override the target. Empty = the base it is parented to, then the nearest. |

**Two traps in that table.**

*The prefab decides the size, the multiplier decides the reservation.* If you override `m_sGroupPrefab` to a bigger group and leave the multiplier alone, the budget pre-flight under-reserves and the wave is released into headroom that cannot hold it — it arrives piecemeal, which reads as bad balance rather than as a decision. PVEF Arland's `Beauregard_Counter_1` does this correctly: `_12` template, multiplier raised to 6, group prefab swapped to `Counter_24`.

*But prefer placing the right template over editing a `_12`.* Every counter on PVEF Arland is a `_12` template with overrides, which means the prefab name in the hierarchy no longer tells you the force size. Place the `_24` when you want 24.

**Waves only advance on a genuine wipe.** A wave that is chased off rather than killed neither ends nor advances, so a counter-attack you walk away from stays live indefinitely. That is deliberate — "spent" is something players earn, not something time delivers — but it also means a group that snags on terrain stands there forever. There is no stuck detection yet.

---

## 7. The mission header

`Missions/<Name>.conf`. PVEF Arland's, verbatim, as the template:

```
SCR_MissionHeaderCampaign {
 World "{D77C9F15B7FA340E}Worlds/PVEF Arland.ent"
 SystemsConfig "{7C9E720397CC6ACD}Configs/Systems/ConflictSystems.conf"
 m_sName "PVEF Arland"
 m_sAuthor "Michael.M"
 m_sDescription "A PvE campaign built on vanilla Conflict. You play US, the enemy is Soviet, and the island gets taken back one base at a time"
 m_sIcon "{319CDD10BD96BA64}Images/PVEF_Arland_Workshop.edds"
 m_sLoadingScreen "{319CDD10BD96BA64}Images/PVEF_Arland_Workshop.edds"
 m_sPreviewImage "{319CDD10BD96BA64}Images/PVEF_Arland_Workshop.edds"
 m_sGameMode "Conflict"
 m_iPlayerCount 128
 m_eEditableGameFlags 6
 m_eDefaultGameFlags 6
 m_bOverrideScenarioTimeAndWeather 1
 m_fNightTimeAcceleration 5
 m_bRandomWeatherChanges 1
}
```

Three notes: it is **`SCR_MissionHeaderCampaign`**, not the plain header; `SystemsConfig` points at **vanilla's** `ConflictSystems.conf`, unchanged; and `m_sGameMode "Conflict"` is what puts it in the right scenario list.

The **scenario ID** a server config needs is the header's own GUID plus path — for PVEF Arland:

```
{1164E83FE2C3C48A}Missions/PVEF_Arland.conf
```

---

## 8. Tuning — every knob is on `PVEF_Manager`

Select the game mode entity in the world and edit `PVEF_Manager`. There is no config file. Defaults shown are the shipped ones.

### Pacing — the four you will actually change

| Attribute | Default | |
|---|---|---|
| `m_iActiveObjectives` | 2 | how many objectives stay open. A **floor**, not a maximum. |
| `m_fRefillDelaySec` | 60 | seconds after a capture before a replacement opens |
| `m_fClusterDistance` | 500 | bases closer than this merge into **one** objective. 0 disables. |
| `m_iMaxClusterSize` | 2 | stops greedy clustering chaining across a dense map |
| `m_fMinSeparation` | 600 | keep separate objectives at least this far apart |

### Map shape

| Attribute | Default | |
|---|---|---|
| `m_iGraphNeighbours` | 3 | lateral links per base on top of the connectivity spine. **Unitless — ports across maps.** 2 plays as a corridor, 4 as a wide front. |
| `m_fRadioRangeScale` | 1.0 | global radio multiplier. Lower it on small islands so coverage does not blanket the map. |
| `m_fGraphMaxLinkDistance` | 0 | optional cap on lateral links, e.g. to sever a link across water. **0 (no cap) is recommended.** |

### AI capacity

| Attribute | Default | |
|---|---|---|
| `m_iAICeiling` | **512** | max simultaneously active AI. Applied at startup and read back. |
| `m_bAssignImportance` | on | **leave this on at 512.** Vanilla ships base compositions at LOW importance, so without this pass the engine sheds objective garrisons *before* roadside ambience. |
| `m_bBudgetReservations` | on | check headroom before releasing a counter-attack wave |

512 is taken from ConflictPVERemixedVanilla2.0's own `SCR_AIWorld` prefab — a measured reference rather than a guess. **`0` is a sentinel meaning "leave the world's value alone" and never reaches the engine**, because the engine reads a ceiling of 0 as permanently full and spawns nothing.

The ceiling is a *capacity*, not a density control. With the governor holding 2 objectives the natural population sits far below it — 56 peak observed on Arland. Raising it does not put more AI on the ground; objective count and counter-attack size do.

### Garrison arming — derived, and mostly leave alone

`m_fGarrisonSpacingFactor` (0.45), `m_fGarrisonMinReach` (150), `m_fGarrisonMaxReach` (600), `m_fGarrisonArmingMultiplier` (1.5), `m_fGarrisonClosingSpeed` (18), `m_fSweepSecondsPerPoint` (1.0), `m_fVanillaSpawnDistance` (600).

The two worth knowing:

- **`m_fGarrisonClosingSpeed` = 18 m/s is road speed in a vehicle.** Raise it for a mission where players routinely arrive by helicopter, or the garrison is still populating when they land.
- **`m_fSweepSecondsPerPoint` is measured from vanilla, not read from source.** If a BI patch changes the ambient system's pacing, re-measure it. The tell is `still queued` becoming common on a front that used to come up cleanly.

### Diagnostics

`m_bLogBaseInventory`, `m_bValidateBases`, `m_bLogGarrisons`, `m_bWatchGarrisons` (1 Hz), `m_bWatchAIBudget` — all on by default. **Leave them on while authoring.** Turn `m_bWatchGarrisons` off on a production server if the log volume is a problem; nothing else depends on it.

`m_iStartupDelayMs` (2000) is how long PVEF waits after world init before scanning, because bases register late. **Do not set it to 0.** If the base inventory comes up empty, raise it.

---

## 9. First boot — the seven lines to read

Run it in Workbench, then read the log from the top. PVEF is loud on purpose.

1. **`Base inventory (N Conflict bases)`** — count them. Empty means the scan ran too early; raise `m_iStartupDelayMs`.
2. **`Expected exactly 2 HQ bases … found N`** — anything but 2 means `m_bCanBeHQ` is true on an island base. Fix that before anything else; the engine will promote one to enemy HQ.
3. **Untagged / duplicate profileId warnings** — every one is a base whose config will attach to the wrong place.
4. **`hq_player` and `hq_opfor_anchor` in the inventory, at the coordinates you expect.** The inference reads faction to tell them apart. If your MOB shows up as `hq_opfor_anchor`, that is the tell — and it fails *quietly*, because both resolve to not-capturable and not-staging, so the round still boots.
5. **Radio reachability report** — names any base that can never be brought into coverage by *any* sequence of captures, with the shortfall in metres and the nearest possible source. A base named here is unreachable for the whole round. Move it, raise a radio range, or place a relay between.
6. **Garrison association** — provenance per spawn point (`remnant` / `child` / `both` / `proximity`), and every unclaimed point named with coordinates and shortfall.
7. **`AI ceiling: X -> 512`** — the read-back proving it took.

Then check the world log for `World doesn't contain RadioManagerEntity`. **Check the play-session log, not just the editor log** — this one has a history of appearing at play time on a world that looked fine in the editor.

### Testing discipline

**Game Master teleporting does not test the loop.** It tests the gate against a movement pattern no player has — one session measured ~790 m covered in seven seconds. It is a fine way to exercise every base quickly, and it is how the arming bug was caught, but it cannot tell you whether the pacing is right. Run both: one Game Master sweep for coverage, one played session end to end for feel.

And when you report a problem: **state how you killed something.** Killing by damage and deleting in Game Master take different code paths, and at least one finding turns entirely on whether the group entity survives its last member.

---

## 10. What PVEF does not do yet

Plan around these — they are gaps, not settings you have missed.

- **No faction lock.** There is no faction config in PVEF Core and the world uses vanilla's `CampaignFactionManager`. Nothing currently stops a player selecting USSR or FIA. If your server needs the player side locked, that is yours to solve for now.
- **No save persistence.** A restart is a fresh round.
- **No road or air patrols.** The ground between the MOB and the front is empty by design right now, and every drive is safe.
- **No artillery, no civilians.**
- **No rank gates or arsenal tiers.**
- **No garbage collection tuning.** At 512 active AI with 48-man waves, corpses and wrecks accumulate fastest exactly where players are. PVEF ships no `SCR_GarbageSystem` config; Gramps ships a tuned one (bodies gone 30 s at 50 m, 300 s at 300 m). Worth adding to your own mission addon in the meantime.
- **No stuck detection on counter-attacks.** Combined with waves only advancing on a wipe, a snagged 48-man group stands still permanently.
- **No terrain-profile generator.** Bases are tagged by hand.

---
