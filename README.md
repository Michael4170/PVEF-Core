<img width="1920" height="1080" alt="PVEFCore_Banner" src="https://github.com/user-attachments/assets/3f4704e9-df8e-40bc-94cc-9e4e368313db" />


PVEF is a PvE framework for Arma Reforger built on vanilla Conflict. Base game only - no companion mod, no dependencies.

Most PvE Conflict servers run a core mod plus a second mod that caps how many objectives are open. That cap is also what stops AI spawning across the whole island. In PVEF the objective governor is the performance mechanism, and it is in the core.

EARLY ACCESS. Working now:

SCENARIOS INCLUDED
- PVEF ARLAND

OBJECTIVE GOVERNOR
- A fixed number of objectives open at once. Two by default, configurable.
- The front is derived from your base layout, not a link radius you tune. Close-together bases merge into one objective.
- Radio coverage is proved at load. A base that can never be reached is named, with the shortfall in metres.
- Objectives refill on a timer after each capture.

AI WHERE THE FIGHT IS
- Garrisons exist only at open objectives, and only once a player is closing on one. The rest of the island is dark and costs nothing.
- Arming distance is derived from live conditions, so defenders are in place before you arrive.
- A base you clear stays cleared.
- PVEF sets the active-AI ceiling itself.

COUNTER-ATTACKS
- Finite waves, each released when the last is wiped.
- They spawn and hold where the map author chose. No random bearing, no beeline for the flag.
- One that cannot be afforded declines rather than half-spawning.
- They stay when you leave. Walking away is not how you beat one.

Reference world: PVEF Arland.

NOT IN YET: patrols, artillery, civilians, save persistence, round rotation, rank gates, Everon map, Kolguyev map, terrain generator, module API docs.

Suggestions and bug reports welcome via [https://discord.gg/SsM7r8b7ae] or through the Github page https://github.com/Michael4170/PVEF-Core.

Credit to Gramps303's ConflictPVERemixedVanilla2.0 and LinearConflictPVE - they set the bar. PVEF is independent - no code or assets from either.

Licence: APL-SA.
