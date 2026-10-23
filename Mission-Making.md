# Building a PVEF mission

How to stand up a new PvE mission on PVEF Core. Written against `PVEF Core` (GUID `6A22BCD5749BFA2B`) and the reference world `PVEF Arland` that ships with it.

**Read this once end to end before you place anything.** Roughly half the steps exist because of a failure that is silent — the mission boots, looks right, and does the wrong thing hours later.

> **PVEF ARLAND IS THE REFERENCE WORLD.** It ships inside PVEF Core and every feature in this document is placed and working in it. The fastest way to build a new map is not to follow this document from the top — it is to **open PVEF Arland alongside your world and copy the pieces you want across**. The "Copying from PVEF Arland" section below is that workflow. Read the rest for what the pieces mean and what to change after pasting.

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

Path B is ~20 base placements of work. The tier prefabs and copying from Arland are what make that a day rather than a week.

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

## 1b. Copying from PVEF Arland — the fast path

Everything below exists, placed and working, in the reference world:

```
PVEF Core/Worlds/PVEF Arland.ent
PVEF Core/Worlds/PVEF Arland_Layers/
    Managers.layer         the six manager entities
    Bases.layer            all 10 capturable bases, with counter-attacks parented under them
    StartingPoints.layer   player MOB + offshore anchor
    AI_Bases.layer         garrison spawn points
    Patrols.layer          roaming patrol spawn points
    Supply_Points.layer    a supply depot with guards and a cache
    World_Edits.layer      terrain and prop edits
```

**Open PVEF Arland in a second World Editor window, select what you want, Ctrl+C, and paste it into your world.**

This is a **Place**, not a Duplicate or an Override: you are copying entity *setups* that reference prefabs by GUID. Nothing of PVEF Core's enters your addon, and you keep getting its updates. It also means a copied base is not a snapshot that goes stale — it still points at `PVEF_Base_Small.et` and inherits any future change to it.

### What to copy for what

| You want | Copy from | Then change |
|---|---|---|
| A capturable base | `Bases.layer` — e.g. `Sm_Base_Mosshill` | **profileId**, display name, coords. Its counter-attacks come with it — re-aim their waypoints. |
| A relay | `Bases.layer` — any `*_Relay` | profileId, name, coords, transmit range if your map is bigger than 4 km |
| Player MOB | `StartingPoints.layer` — `PVEF_Base_Mob` | coords only |
| Offshore anchor | `StartingPoints.layer` — `PVEF_Base_USSR` | coords — **must be in water, off the playable area** |
| A counter-attack on its own | any base's child in `Bases.layer` | parent it to the new base, move the defend waypoint onto the ground to hold |
| A garrison | `AI_Bases.layer` | parent it to the new base |
| A roaming patrol | `Patrols.layer` | coords. The `PVEF_RoamingPatrolTag` comes with it. |
| A supply point | `Supply_Points.layer` — `Timberridge_Supply_Depot` | coords. Guards, roaming tags and the cache all come with it. |
| The manager set | `Managers.layer` — all six entities | **the navmesh references — see below** |

### The five things that do not survive the copy

Every one of these fails silently if you skip it.

1. **Navmesh references, if you copied the Managers layer.** `SCR_AIWorld`'s three `NavmeshWorldComponent`s still point at Arland's meshes. Nothing errors; AI simply cannot path, which reads as broken AI rather than missing config. **Re-point them first** — see the navmesh section.

2. **`m_sProfileId` must be unique.** A copied base arrives carrying Arland's key. Two bases with the same profileId means per-base config silently attaches to the wrong one. The validator warns about duplicates when the world loads.

3. **Defend waypoints keep a LOCAL offset, not a world position.** A parented waypoint serialises as an offset from its parent, so a copied counter-attack lands its objective the same distance and bearing from the spawn point as it did on Arland — which will be somewhere arbitrary on your site. Re-drag every one.

4. **Radio ranges are sized for a 4 km island.** Arland relays transmit 1000 m. On a bigger map either raise them per relay or use `m_fRadioRangeScale` on `PVEF_Manager` to scale the lot.

5. **`m_iActiveObjectives` is 1 on Arland deliberately**, because it is small. It is on the `PVEF_Manager` instance, so it comes across if you copy the game mode entity. Most maps want the shipped default of 2, or 3 for Everon scale.

---

## 2. The world, and the six entities that must be in it

Create your world as a sub-scene of the terrain world, then make a **`Managers` layer** and put these six in it. This list is copied from `PVEF Arland_Layers/Managers.layer` — it is the complete set, and most of the six fail *silently* if missing.

| Entity | Prefab | Missing = |
|---|---|---|
| `SCR_AIWorld` | `{E0A05C76552E7F58}Prefabs/AI/SCR_AIWorld.et` | AI cannot path anywhere. Looks like broken AI, not missing config. |
| `PerceptionManager` | `{028DAEAD63E056BE}Prefabs/World/Game/PerceptionManager.et` | AI cannot see. |
| **Game mode** | `{ABE4F8FEE9FCD1E2}Prefabs/MP/Modes/Conflict/GameMode_PVEF_Conflict.et` | no framework at all |
| `RadioManagerEntity` | `{B8E09FAB91C4ECCD}Prefabs/Systems/Radio/RadioManager.et` | radio coverage never works; base connection lines never draw |
| `SCR_CampaignFactionManager` | `{F1AC26310BAE3788}Prefabs/MP/Campaign/CampaignFactionManager.et` | no factions |
| `SCR_LoadoutManager` | `{58FBD035E53D95C1}Prefabs/MP/Campaign/CampaignLoadoutManager.et` | no loadouts |

`GameMode_PVEF_Conflict.et` already inherits vanilla `GameMode_Campaign.et` and already carries `PVEF_Manager`. **Do not add `PVEF_Manager` yourself** — one per world, and it is in the prefab.

> **PVEF Core OVERRIDES the faction manager at that vanilla GUID.** That is how the faction lock works (USSR and FIA are set non-playable), and it is why you place the vanilla path and still get PvE behaviour. Be aware of the consequence: the override applies to **every** Conflict scenario on any server running PVEF Core, not just PVEF missions. Do not ship PVEF Core in a mod set alongside a PvP mod that needs a playable OPFOR.

On the placed instance you set the victory threshold. PVEF Arland uses:

```
m_iControlPointsThreshold 10
```

That is "players hold 10 control points and the round ends". Count your capturable bases and pick a number below that — 10 of 10 on Arland means taking the island.

### Layers

Organise by **system**, not per base. PVEF Arland's seven layers are listed above, and copying that structure is the easiest way to get it right. You can toggle a whole subsystem while editing, and related entities stay together.

---

## 3. Navmesh — and the shortcut that is legitimate

AI pathing needs `SCR_AIWorld`'s `NavmeshWorldComponent`s to *point at* `.nmn` files. Generating a navmesh is not enough; if nothing points at it the log says *"No navmesh file specified! Will initialize empty navmesh world"* and every AI stands still.

**If your terrain is unmodified vanilla, reuse BI's navmesh instead of generating one.** PVEF Arland does exactly this — three `NavmeshWorldComponent`s pointing at:

```
{D8EF7131FB31AF97}worlds/GameMaster/Navmeshes/GM_Arland.nmn          soldiers
{A0AEEB1E7EF474FA}worlds/GameMaster/Navmeshes/GM_Arland_vehicles.nmn vehicles
{804386FFEB7B7EFD}worlds/MP/Navmeshes/LowResArland.nmn               low-res
```

The Game Master navmeshes cover the whole terrain and BI maintains them. This works because path B changes no terrain — you place buildings *on* the existing heightmap. Find the equivalent three for your terrain under `worlds/GameMaster/Navmeshes/` and `worlds/MP/Navmeshes/`.

> **COPYING ARLAND'S MANAGERS LAYER BRINGS ARLAND'S NAVMESH WITH IT.** Its three navmesh references still point at `GM_Arland.nmn`, wherever your world actually is. Nothing errors. AI simply cannot path. **If you copy the Managers layer, re-point the navmeshes before anything else** — it is the single most expensive copy mistake to get wrong, because it presents as the framework being broken.

**You must generate your own the moment you change the ground.** Flattening under a base, Terrain Tools → Bake Selection, or placing a composition that cuts into a slope all invalidate the baked mesh locally. Then: Navmesh Tool → Connect → Generate (or *Rebuild changed tiles*), and **tick "Autosave when done" or click Save afterwards** — without that the generated mesh is discarded and you have done nothing.

---

## 4. Bases

### The tier prefabs

PVEF Core ships five. Place these, not vanilla base prefabs — they already carry `PVEF_BaseTag` with the right tier and the right `SCR_CampaignMilitaryBaseComponent` flags. **Or copy a configured one out of Arland's `Bases.layer`, which saves setting the flags at all.**

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

**Why the profileId is mandatory and why it must not be the display name.** Conflict randomises in-game base names every session — the same base reads as CHICAGO one run and COMET the next. `m_sProfileId` is the stable key everything else attaches to. Get it wrong and per-base config silently attaches to the wrong base after a restart, which is the kind of bug that takes a week to see. **This is also the first thing to change on any base copied from Arland.**

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

1000 m on a 4 km island. On a bigger map, raise the ranges or place more relays — and see `m_fRadioRangeScale` under Tuning for the global multiplier that saves you editing every base.

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

Use `AmbientPatrolSpawnpoint_USSR.et` (`{A73205DEA8361F26}`), OPFOR-affiliated from birth — one less runtime mutation than the FIA variant vanilla bases ship with. Arland's are in `AI_Bases.layer` if you would rather copy a working one.

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

### Garrisons that must not move

A garrison placed in a cave, bunker or room — where a normal patrol would wander out or chase gunfire down the road — should be a **hold-position garrison** rather than a patrol with a short leash. They are placed the same way, one point per soldier; see section 6d.

### Roaming patrols

A patrol that should stay always-on and never go dark with an objective gets `PVEF_RoamingPatrolTag` on the spawn point, plus `m_eImportance = LOW` so it yields budget to real fights. A patrol placed well away from every base needs no tag — nothing is in range to claim it. Arland's live in `Patrols.layer`.

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

1. Drag the template into the world — or copy a configured one out of Arland's `Bases.layer`.
2. **Parent it to the base it attacks.**
3. Move its child defend waypoint onto the ground you want held.

Place it where the attack should come *from*. By default every wave spawns exactly there — you picked that spot by reading terrain, which beats five procedural tests. If you want waves to come in from slightly different places each time, turn on spawn scatter (below).

### The one thing that will bite you

**A nested entity needs a `Hierarchy` component or it is never a runtime child.** Without it the waypoint exists and is correctly placed, but the counter-attack never sees it. The shipped templates have it, and so does anything copied from Arland. If you ever build a spawn point from scratch, this is the thing to get right, and the "no waypoint parented" warning names it explicitly.

### Large counters spawn as groups of 12

A `_24` or `_48` counter is still **one counter** — one spawn point, one defend waypoint, one budget reservation, one wipe to advance the wave — but its infantry is fielded as groups of **at most 12**, because one big group bogs down when it reaches a single defend marker. Nothing to place or set: you still place only the one defend waypoint, and the extra groups find their own positions around it. It is fixed behaviour with no setting, so it applies to every infantry counter on the server.

### Tuning a placed counter-attack

| Attribute | Default | What it does |
|---|---|---|
| `m_iRespawns` | 2 | respawns **after** the first wave. `2` = three waves total. `-1` = endless. |
| `m_iRespawnSeconds` | 30 | gap after a wave is wiped before the next is released. `0` = framework default (15 s). |
| `m_fObjectiveRadius` | 100 | how much ground the force holds. **15 m makes twelve men stand on one spot.** |
| `m_fPlayerClearRadius` | 0 → 212 m | how close a player must be to *hold* a wave. Not lost — it fires on a later pass. |
| `m_fSpawnScatterRadius` | 0 | spawn scatter, in metres. `0` = off — every wave spawns where you placed it. See below. |
| `m_sTargetBaseId` | empty | override the target. Empty = the base it is parented to, then the nearest. |

**The force size comes from the template you place.** `m_iGroupMultiplier` and `m_sGroupPrefab` still exist on `PVEF_CounterAttackTag` but are hidden in the editor, so they cannot be changed from there. Place `_12`, `_24` or `_48` for the size you want; the template carries the matching group prefab and budget reservation.

### Spawn scatter (optional)

Set `m_fSpawnScatterRadius` above 0 and **each wave spawns at a random point within that radius** of the placed spawn point, so players cannot pre-aim one treeline. The defend waypoint does not move.

Every rolled point is checked before a wave spawns there:

- above sea level (inland ponds and lakes are **not** checked);
- ground no steeper than about 30°;
- clear of rocks, fences and buildings;
- no player inside the clear radius of the new spot;
- no more than halfway from the placed spot to the objective;
- **at least 100 m from the base and from the defend waypoint.**

It gets three tries per wave. If none pass, the wave spawns at your placed spot, exactly as it would with scatter off.

**Suggested radius:** about 15–25% of the counter's distance to its objective — **30–50 m** for a small shift along the same approach, **75–100 m** for a different angle in open ground. Keep the radius inside ground you would happily have placed the spawn point on: the checks catch slopes and objects, not whether the AI can walk out of a boxed-in dip. If a roll does land somewhere the wave cannot leave, the stuck detector gives up on it and logs where it stopped — shrink that counter's radius.

The log shows each scattered wave:

```
Counter-attack [x, y] wave 2 scattered 107m from its placed position, to [x, y].
```

The rolls are spread evenly over the whole circle, so the average wave lands about two-thirds of the radius out and few land right at the edge. A 150 m radius typically gives 50–140 m.

Scatter costs nothing measurable: it runs once per wave, at spawn, and never per tick. The server-wide `m_fSpawnScatterScale` on `PVEF_Manager` multiplies every radius — see Tuning.

### Waves, and what happens when one does not arrive

**A counter-attack only spawns once its base is an active objective.** Players being in range of an inactive base never triggers one. Once triggered, a counter-attack walks to its defend waypoint and stays live through its waves.

**Waves only advance on a genuine wipe.** A wave that is chased off rather than killed neither ends nor advances, so a counter-attack you walk away from stays live indefinitely. That is deliberate — "spent" is something players earn, not something time delivers.

**A wave that is genuinely stuck is given up on.** A counter-attack never gives up otherwise. If a wave closes less than 25 m in 90 seconds, is more than 50 m from its objective, is at full strength and is not taking casualties, PVEF deletes **that wave only**. Two log lines matter:

```
Counter-attack [x, y] GIVEN UP ON - wave 1 closed only 2m in 90s and is still 582m from its objective, 12 alive and unengaged.
  Stopped around [x, y]. THE FIX IS TO MOVE THIS COUNTER'S SPAWN POINT somewhere its force can path out of -
Counter-attack: counter-attack at [x, y] is DONE after 1 wave(s) - GIVEN UP ON, it never reached its objective.
```

**Read that as an authoring fault, not a bug.** The position is authored, so a replacement wave would walk onto the same ground. The coordinates in the second line are where the force could not path out of. Move the spawn point.

This is also the most likely thing to fire on a **freshly copied** counter-attack, because a pasted defend waypoint keeps its old local offset and may be pointing at ground the force cannot reach.

Two consequences worth knowing before you file a bug report:

- **Only the stuck wave is given up on, not the whole counter.** A counter that has been given up on still counts as finished and stays finished across a server restart, like any other finished counter-attack.
- **A wave in a firefight is never given up on.** The check resets whenever the headcount changes, so a wave taking casualties or still populating is left alone however little ground it covers.

If you see `GIVEN UP ON` with a **full alive count** on a wave that was plainly in contact, that *is* a bug — report it with the log lines.

### When a counter-attack takes a base back — retake objectives

If a counter-attack takes back a base the players had captured, PVEF opens it again as a **retake objective**, with its own task marker, so players can see it needs taking back. There is nothing to place or configure — it works on any map.

- **It opens straight away.** No refill delay.
- **It does not count toward `m_iActiveObjectives`.** The normal front keeps its slots, so a retake is extra. It sits outside the normal objective sequence and opens alongside the active objective, so the task players are already working on stays open.
- **Winning it back opens nothing new.** The retake simply closes.
- **The base's garrison stays off.** The counter-attack force that took it is the defence.
- **The MOB and the offshore anchor are never retakes.**
- **Half of a clustered objective** that is lost while the objective is still open is left alone — it already has its task.

The log names each one:

```
Governor: Mosshill LOST to USSR - opened as a retake objective.
  RETAKE  Mosshill  (outside the objective count, no garrison)
Governor: retake Mosshill WON BACK.
```

A retake marker only shows while the base is inside radio coverage, like any other seize task. If losing a relay pushes the base out of coverage, the marker appears once coverage comes back.

When a base is lost, front objectives that are still inside radio coverage keep their task markers, so players can see those points are still active. Only objectives that have genuinely gone out of radio range lose their task, exactly as in vanilla.

---

## 6b. Supply points

Supply points are pure vanilla content placed in a layer. **PVEF has no supply code and there is nothing to tag** — they are not bases, they never register with the governor, and they cost no AI budget on their own.

The fastest route is to copy `Timberridge_Supply_Depot` out of Arland's `Supply_Points.layer` and move it: the guards, their roaming tags and the cache all come with it, and only the coordinates need changing.

### The two things you can place

| What | Prefab | Use |
|---|---|---|
| **Depot** | `{27941CDF3E7E1A60}Prefabs/MP/Campaign/CampaignRemnantsSupplyDepot.et` | a supply objective with a **map icon**. This is what you want if the point should appear on the map. |
| **Cache** | `Prefabs/Compositions/Slotted/SlotFlatSmall/SupplyCache_S_FIA_01.et` … `_06.et` | loose supplies lying around. No map presence, no AI. |

Cache GUIDs, all six:

```
{AB1A97B1BAE8C395} _01    {C74EA351062C4CBB} _02    {22D2EFA80AC9DBD0} _03
{1FE6CA907FA552E7} _04    {FA7A86697340C58C} _05    {962EB289CF844AA2} _06
```

**Do not use `Prefabs/Compositions/Locations/Eden/SupplyCache_<Town>_FIA_01.et`.** "Eden" is Everon's internal name and those are BI's hand-built caches for specific Everon towns. They will place anywhere and belong nowhere else.

### The gotcha — caches ship with resource gain OFF

Every cache instance needs this override or it is a decorative pile of crates that generates nothing, with no error and no log line:

```
SCR_ResourceComponent {
 m_aContainers {
  SCR_ResourceContainerVirtual {
   m_fResourceValueCurrent 1000
   m_fResourceValueMax     1000
   m_bEnableResourceGain   1
  }
 }
}
```

Same family as `_NotSpawned` group prefabs: the thing looks placed and correct and does nothing. If you are placing more than a couple, inherit a cache prefab with the values already set and place that — PVEF Arland uses `Prefabs/MP/Campaign/SupplyDepots/SupplyCache_S_FIA_PVEF.et` for exactly this, and copying the depot brings it along.

### Guards on a depot must be tagged

A depot is not an objective, so its defenders must not be gated like a garrison. Parent `AmbientPatrolSpawnpoint_USSR.et` under the depot and put **`PVEF_RoamingPatrolTag`** on each one. Without the tag:

- a depot near a base has its guards adopted as that base's garrison and gated with it;
- a depot away from every base leaves them **unclaimed**, never gated, holding ambient-rotation slots all round and inflating the arming radius for every real garrison on the map.

Consider `m_eImportance = LOW` as well, so depot guards yield AI budget to objective fights. NORMAL is defensible if you want the depot properly held — make it a decision rather than a default.

### Do not try to change the map icon colour

The depot's icon is green and **there is no authoring route to change it.** None of these work, and all fail silently:

1. `SCR_MapDescriptorComponent`'s own `Faction` field — setting it to 2 changed nothing, not even to blue, which is what 2 means.
2. Adding `SCR_FactionAffiliationComponent` set to USSR — no effect.
3. Calling `MapItem.SetFactionIndex()` directly from a script component — no effect.
4. Overriding the icon asset.

**The reason:** colour is a property of the *(descriptor type, faction)* pair held on the `MapLayer`, not on the entity. `MapLayer.GetPropsFor(iFaction, type)` is where it comes from, and `Icon (generic)` evidently renders the same colour across all three faction indices. Bases are red because they carry `SCR_CampaignMilitaryBaseMapDescriptorComponent`, a subclass whose `MapSetup(Faction)` runs at runtime — the plain descriptor on a depot has no such call.

If you need supply points to read differently on the map, use **`Display Name`** on the descriptor and a non-generic **`Main Type`**. A labelled icon of a different shape separates them from bases better than colour would, and costs nothing.

---

## 6c. Mortars

A mortar position is a crewed, emplaced tube that shells ground near a base. **It stays quiet until that base's counter-attacks are finished**, so it is the next stage of the fight for a base rather than something that opens on first contact. Once they are finished it fires regardless of whether the base has since been taken or is still an open objective, and that holds across a server restart. Turn the whole feature on or off with `m_bMortars` on `PVEF_Manager` (on by default).

### Place it in three steps

1. Place `AmbientPatrolSpawnpoint_USSR_Mortar` where the **tube** should sit.
2. **Parent it to the base it belongs to.**
3. Drag its child waypoint onto the ground you want **shelled**. That waypoint is the impact point.

The template already names the tube, the crew and the fire-mission waypoint, so there is nothing else to set for a working position.

### How far the tube can be from its waypoint

**At least 25 m, at most 1000 m.** Closer than 25 m is rejected with a warning, on the assumption the waypoint was never dragged off the template. Past 1000 m the aim point is quietly pulled back to 1000 m along the bearing, so every round lands short. At longer ranges the crew fires on a stronger charge automatically; there is nothing to set.

Rounds scatter around the waypoint, so for the whole spread to be reachable, keep the tube inside these distances:

| Beaten zone | Max tube-to-waypoint distance |
|---|---|
| 150 m | 850 m |
| 100 m | 900 m |
| 80 m | 920 m |
| 60 m | 940 m |
| 35 m (the minimum) | 965 m |

Leave some margin — with a 150 m zone, about 800 m is a sensible placement.

### Tuning a placed mortar — `PVEF_MortarTag`

| Attribute | Default | What it does |
|---|---|---|
| `m_fBeatenZone` | 0 | how far rounds scatter around the waypoint. `0` uses the waypoint's own completion radius (the circle you see in the editor). Never less than 35 m. |
| `m_fPlayerPresenceRadius` | 300 | a player must be this close to the **impact point** for the tube to fire. Also what quiets it once the front moves on. |
| `m_fPlayerClearRadius` | 0 → 212 m | how close a player must be to *hold* the crew spawn. Nothing is lost — it spawns once they move off. |
| `m_fSpawnScatterRadius` | 0 | spawn scatter, in metres. `0` = off — the tube goes where you placed it. See below. |
| `m_iCrewReplacements` | 1 | how many times a killed crew is replaced. `0` = one crew only. |
| `m_iRespawnSeconds` | 0 → 15 s | gap before a replacement crew arrives |
| `m_iGroupMultiplier` | 2 | sizing figure for the crew's budget entry, in fours. **Does not resize the crew.** Mortar crews are exempt from the AI budget limit. |
| `m_sTargetBaseId` | empty | override the base. Empty = the base it is parented to, then the nearest. |

**Destroying the tube ends the position for the round** — the tube is never respawned, only the crew.

Rounds go out in groups of four, and the gap between groups is the crew re-laying the gun. `m_iShotsPerMission`, `m_iSecondsBetweenMissions` and `m_iSecondsBetweenRounds` do not change the rate of fire.

### Spawn scatter (optional)

Set `m_fSpawnScatterRadius` above 0 and the **tube position is picked at random within that radius, once per round**, so it is not in the same pit every session. Replacement crews go to wherever it landed. The impact waypoint does not move, and the tube still turns to face it from its new spot — the whole composition turns with it, sandbags included.

The same ground checks as counter-attacks apply (sea, slope, objects), plus:

- **never further** from the impact point than where you placed it, and not much closer — between 60% and 100% of the placed distance;
- never within 110 m of the impact point;
- **at least 100 m from the base it is parented to.**

Three tries; if none pass, the tube goes where you placed it. **30–50 m is enough.** If the tube is part of a composition with sandbags, keep it small — the whole composition moves. The log shows it at startup:

```
Mortar [x, y] scattered 27m for this round, to [x, y] - 315m from its impact point (placed 337m).
```

### Reading the log

```
Mortar: mortar at [x, y] burst 1 OPEN - 4 round(s) onto [x, y], 798m out, ... Solve: charge 3 of 5, ...
```

`798m out` is the range actually being fired at, and `charge 3 of 5` is the charge the crew is using for it. If the range reads **1000** every time, the tube is too far from its waypoint — move it closer. `has NO child waypoint` or `impact point only Nm away` means the waypoint was never placed or never dragged off the template.

---

## 6d. Hold-position garrisons

A hold-position garrison is one soldier placed to guard one exact spot — a cave mouth, a bunker, a room — who **cannot walk away from it**. He still sees, turns, aims and fires; players have to go in and clear him. Use it where an ordinary garrison would wander out of the door or chase gunfire down the road, which is exactly what a normal patrol does inside a building.

It is the same machinery as any garrison — an ambient patrol spawn point — so it spawns, is budgeted and (optionally) is gated like one. Only the movement is different.

### Place it in three steps

1. Place **`AmbientPatrolSpawnpoint_USSR_Hold`** (`{8B42C6D1F0A97E35}`, under `Prefabs/Systems/AmbientPatrol/`) **where the soldier should stand.** In the editor it draws a purple person-shaped marker, and he spawns on that spot. **One point is one soldier**, so you place each man exactly.
2. Decide how it is gated — see the next section. A guard on a base objective is **parented to that base**; a cave or building that is not part of an objective gets **Always Active** ticked.
3. Optionally set a stance.

To field a pair from one point, set the group prefab to `PVEF_Group_USSR_Hold_2` (`{5D1F3A7C9E24B860}`). Two men from one point stand on the same spot, so one point per soldier is the way to place them precisely.

### Attributes — `PVEF_HoldPositionTag`

| Attribute | Default | What it does |
|---|---|---|
| `m_bEnabled` | on | untick to hand the point back to normal behaviour without deleting the component |
| `m_eStance` | Free | `Free` leaves stance to the AI. `Stand`, `Crouch` or `Prone` limits him to that stance — prone or crouched suits a doorway guard. |
| `m_bAlwaysActive` | off | ignore base objectives — see below |
| `m_sGroupPrefab` | `PVEF_Group_USSR_Hold_1` (`{6E2B9D41A8C37F05}`, one soldier) | what is fielded. **Empty falls back to the faction catalog, which fields a full patrol-sized squad** — almost never what you want for a guard. |

### Gating — and the two ways a hold point goes dark

By default a hold point is gated **exactly like any garrison**: near a capturable base it is claimed by that base and spawns only while that base is an active objective. Parenting it to the base makes that explicit and works at any range.

Both of these fail silently — no error, the soldier simply never appears:

1. **Placed within reach of the player MOB or the offshore anchor.** The nearest base claims it, and PVEF never spawns anything at either of those. It stays dark all round. Tick Always Active, or move it.
2. **Placed in a cave away from every objective.** It is claimed by whichever base is nearest, which may be the wrong one.

**Always Active** is the remedy for both. It reserves the point from every base pass, the same way `PVEF_RoamingPatrolTag` does: no base claims it and no objective gates it, so vanilla's own ambient distance rules decide when it is live. It still holds a slot in the ambient rotation like a roaming patrol, so do not scatter hundreds.

> **An Always Active point is also protected from a base capture.** Vanilla retires a base's registered ambient points when its faction changes, and a point near a base can be registered to it. PVEF removes an Always Active hold point from every base's list at startup and revives it if it has already been retired — you will see `it had been retired before PVEF saw it - revived` in the log when that happens. This is the fault that kept the first hold points from ever spawning, and nothing in the log says so unless you look for it.

### What "cannot walk" means

A hold soldier stands where he spawned. He turns toward threats and fires, he does not investigate gunfire, flank, take cover or retreat, and he does not step out of the door. He does not dodge grenades or flee, which makes him easy to clear — that is part of the design.

**Facing is not controllable.** Vanilla spawns the group at the spawn point's *position only*; rotating the entity does not turn the soldier. He starts with a default heading and turns once he sees something. Place him where the *position* guards the approach, not where his initial facing does.

The group spawns early in the round — the logs show roughly 7–10 seconds after the world starts — and is then activated or deactivated by player distance like any ambient patrol.

**Respawn follows the spawn point's own `m_iRespawnPeriod`.** The shipped default is `0`, so a cleared hold point stays cleared.

### Reading the log

```
hold-position point at [x, z] -> <group prefab> (ALWAYS ACTIVE, ungated)     startup: found, and how it is gated
    it had been retired before PVEF saw it - revived.                       startup: see the note above
Hold position at [x, z]: SpawnPatrol call #1, prefab=..., group fielded=yes. spawned
Hold position at [x, z]: AI movement settings locked (IDLE), stance FREE.    lock layer 1 applied
Hold position: member 0 first pinned at [x, z].                               his body exists and is pinned
Hold position at [x, z]: pinned 1 member(s) to zero speed, waypoints removed. lock layers 2 and 3 applied
```

**`group fielded=NO` with a prefab named** means the group prefab failed to spawn. **No `SpawnPatrol call` line at all** means the ambient system never visited the point — read the two `Hold probe` lines that follow six and twenty seconds after startup: they report whether the point is registered, spawned, eliminated, paused or has a respawn pending. For an Always Active point the probe also fixes a point that is missing from the rotation or flagged eliminated, and says so.

A line like this means the lock leaked and should be reported with the surrounding log:

```
Hold position: member 0 HAS MOVED 4.5 m from where it was pinned (now [x, z]).
```

A run with no such line means he held.

### For maintainers — why the lock has three layers

It is tempting to simplify this to the AI settings system, which has a movement-speed limit. **Do not.** Limiting the AI's speed to `IDLE` was tried first and **did not stop the soldiers walking out of the building.** The lock is three layers, applied by `PVEF_HoldPositionTag` and re-applied every 1.5 seconds for the life of the group by `PVEF_ModdedHoldPosition`:

1. **AI settings** — group and character movement-speed limited to `IDLE`, plus the optional stance limit. Kept, but not trusted alone.
2. **No waypoint** — vanilla gives every ambient group a Defend waypoint, and Defend moves the AI between cover positions around it, which is what carried them out of the door. The hold removes every waypoint from the group.
3. **Character speed pinned to zero** — `CharacterControllerComponent.OverrideMaxSpeed(0)` acts on the character itself rather than on the AI's decisions, so it holds whatever the AI wants. Aiming, turning and firing are separate from locomotion and keep working.

It repeats because members are created after `SpawnPatrol()` returns and the engine can reset a speed override.

Two API facts that cost a compile cycle: `GetMembersAlive()` and `SetMembersAlive()` **do not exist in 1.8.0.10** even though they appear in 1.7 source, and a single undefined function fails the whole module's compile.

### Verified in play, and not yet

**Verified on PVEF Arland:** four Always Active hold points placed in a cluster, each fielded one soldier, each was pinned to zero speed within about five seconds of spawning, and a four-minute run logged no `HAS MOVED` line. The startup revive fired and fixed two of the four.

**Not yet exercised:** a hold point gated by a real base objective (parented, not Always Active); the `Stand`, `Crouch` and `Prone` stances; a non-zero `m_iRespawnPeriod`; and a large number of hold points at once. Treat those as unproven until a run covers them.

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

Persistence rides on that same vanilla systems config — PVEF ships one small override at `Configs/Systems/Persistence/GameMode/Conflict.conf` and nothing else. **Finished counter-attacks and mortars stay finished across a server restart**, so clearing a base's counter-attacks and restarting does not bring them back when the base is retaken. There is no file to edit and nothing to configure. They reset when the mission ends, however it ends, and when the Conflict save is deleted to start the map over. **A Workbench play session always starts with every counter-attack and mortar armed**, so you never need to clear any saved state while authoring. PVEF Core's counter-attack and mortar state is kept inside the Conflict save, so there are no extra files in the server profile. A mortar that has not yet spawned always carries on after a restart; one that has already played out does not come back. The rest of PVEF's own state — the governor's open objectives and the civilian latch — is not serialised and comes back at defaults after a reload. Base ownership, which is what players notice, is vanilla's and does persist.

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

Retake objectives (see Counter-attacks) sit outside all of these: they do not count toward `m_iActiveObjectives` and do not wait for `m_fRefillDelaySec`.

**Objective count follows map size and travel time, not the default.** PVEF Arland runs `m_iActiveObjectives 1` — deliberately, because a 4 km island with two objectives open gives no travel and no front. Everon-scale maps want 2 or 3. The shipped default of 2 is the normal case; small islands are the exception. **Copying Arland's game mode entity brings the 1 with it.**

`m_fRefillDelaySec 60` is the tested value. Longer delays leave players with nothing to do between objectives.

### Spawn scatter

| Attribute | Default | |
|---|---|---|
| `m_fSpawnScatterScale` | 1 | multiplies every counter-attack and mortar `m_fSpawnScatterRadius`. `0` = scatter off server-wide, `1` = each spawn point's own radius, `2` = double. Spawn points left at radius 0 never scatter, whatever this says. |

You do not need to touch this to use scatter — setting a spawn point's radius is enough. It is there to turn scatter down or off for a whole server without editing every placement.

### Map shape

| Attribute | Default | |
|---|---|---|
| `m_iGraphNeighbours` | 3 | lateral links per base on top of the connectivity spine. **Unitless — ports across maps.** 2 plays as a corridor, 4 as a wide front. |
| `m_fRadioRangeScale` | 1.0 | global radio multiplier. Lower it on small islands so coverage does not blanket the map; raise it on large ones rather than editing every relay. |
| `m_fGraphMaxLinkDistance` | 0 | optional cap on lateral links, e.g. to sever a link across water. **0 (no cap) is recommended.** |

### AI capacity

| Attribute | Default | |
|---|---|---|
| `m_iAICeiling` | **512** | max simultaneously active AI. Applied at startup and read back. |
| `m_bAssignImportance` | on | **leave this on at 512.** Vanilla ships base compositions at LOW importance, so without this pass the engine sheds objective garrisons *before* roadside ambience. |
| `m_bBudgetReservations` | on | check headroom before releasing a counter-attack wave |

512 is a measured reference value rather than a guess. **`0` is a sentinel meaning "leave the world's value alone" and never reaches the engine**, because the engine reads a ceiling of 0 as permanently full and spawns nothing.

The ceiling is a *capacity*, not a density control. With the governor holding a small objective count the natural population sits far below it — 56 peak observed on Arland. Raising it does not put more AI on the ground; objective count and counter-attack size do.

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

Run it in Workbench, then read the log from the top. PVEF is loud on purpose. **On a world built by copying from Arland, lines 1, 3 and 5 are the ones that catch what the copy brought with it.**

1. **`Base inventory (N Conflict bases)`** — count them. Empty means the scan ran too early; raise `m_iStartupDelayMs`.
2. **`Expected exactly 2 HQ bases … found N`** — anything but 2 means `m_bCanBeHQ` is true on an island base. Fix that before anything else; the engine will promote one to enemy HQ.
3. **Untagged / duplicate profileId warnings** — every one is a base whose config will attach to the wrong place. **Duplicates are the classic copy-from-Arland fault.**
4. **`hq_player` and `hq_opfor_anchor` in the inventory, at the coordinates you expect.** The inference reads faction to tell them apart. If your MOB shows up as `hq_opfor_anchor`, that is the tell — and it fails *quietly*, because both resolve to not-capturable and not-staging, so the round still boots.
5. **Radio reachability report** — names any base that can never be brought into coverage by *any* sequence of captures, with the shortfall in metres and the nearest possible source. A base named here is unreachable for the whole round. Move it, raise a radio range, or place a relay between. **On a bigger map than Arland this is where 1000 m relay ranges show up as too short.**
6. **Garrison association** — provenance per spawn point (`remnant` / `child` / `both` / `proximity`), and every unclaimed point named with coordinates and shortfall.
7. **`AI ceiling: X -> 512`** — the read-back proving it took.

Then check the world log for `World doesn't contain RadioManagerEntity`. On PVEF Arland this fires ~8 seconds before PVEF's startup block and is an **ordering artifact, not an absence** — the manager is present and radio works. Do not chase it.

### Testing discipline

**Game Master teleporting does not test the loop.** It tests the gate against a movement pattern no player has — one session measured ~790 m covered in seven seconds. It is a fine way to exercise every base quickly, but it cannot tell you whether the pacing is right. Run both: one Game Master sweep for coverage, one played session end to end for feel.

And when you report a problem: **state how you killed something.** Killing by damage and deleting in Game Master take different code paths, and they can behave differently.

---

### What is included

All of these work, and all of them are placed and working in PVEF Arland:

**faction lock** (USSR and FIA non-playable, via PVEF Core's faction manager override) · **AI seizing** (the enemy takes bases back, it does not just defend) · **civilians** in towns the round has reached · **garbage collection** of bodies and wrecks on a shorter clock than vanilla near players · **save persistence** for vanilla state · **counter-attack stuck detection** · **supply points** · **retake objectives** — a base lost to a counter-attack gets its task marker back · **mortars** · **optional spawn scatter** for counter-attacks and mortars · **hold-position garrisons** — soldiers who guard one spot and do not leave it.

---

## Reference: PVEF Arland at a glance

The reference world, and the thing to copy from. Also useful as a sanity check on your own numbers.

| | |
|---|---|
| World | `PVEF Core/Worlds/PVEF Arland.ent` |
| Layers | `PVEF Core/Worlds/PVEF Arland_Layers/` — seven, listed under Copying from PVEF Arland |
| Mission header | `PVEF Core/Missions/PVEF_Arland.conf` |
| Terrain | Arland (~4 km), path B sub-scene |
| Bases | 5 RELAY, 2 LARGE, 3 SMALL, + MOB + offshore anchor |
| Relay transmit range | 1000 m each |
| Victory threshold | 10 control points |
| Objectives open | **1** (small map — see Tuning), 60 s refill, cluster 500 m / max 2 |
| Graph | 3 lateral neighbours, no link cap |
| AI ceiling | 512 |
| Counter-attacks | 13 placed, all `_12` templates with overrides |
| Mortars | 2, both shelling Arleville |
| Supply points | 1 depot (`Timberridge_Supply_Depot`) with 2 roaming-tagged guards and an inherited cache |
| Navmesh | BI's Game Master meshes, reused |
| Peak AI observed | 56, with a clustered objective open |
