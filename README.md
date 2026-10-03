<img width="1920" height="1080" alt="PVEFCore_Banner" src="https://github.com/user-attachments/assets/3f4704e9-df8e-40bc-94cc-9e4e368313db" />


PVEF is a PvE framework for Arma Reforger built on vanilla Conflict. Base game only - no dependencies.

ALPHA, for testing. Working now:

OBJECTIVE GOVERNOR
- A fixed number of objectives open at once. Two by default, configurable.
- The front is derived from your base layout, not a radius you tune. Close-together bases merge into one objective.
- Radio coverage is proved at load. An unreachable base is named, with the shortfall in metres.

AI WHERE THE FIGHT IS
- Garrisons exist only at open objectives, and only once a player closes on one. The rest of the island is dark and costs nothing.
- A base you clear stays cleared. PVEF sets the active-AI ceiling itself.
- Bodies and wrecks are cleared faster than vanilla.

COUNTER-ATTACKS
- Finite waves, each released when the last is wiped.
- They spawn and hold where the map author chose, with optional scatter. No beeline for the flag.
- One that cannot be afforded declines rather than half-spawning.
- They stay when you leave.
- A wave that stops making progress is given up on, so a badly sited counter cannot hold AI all round. The log names the spot.

ALSO WORKING
- The enemy side is locked - players cannot join USSR or FIA.
- The enemy takes bases back, it does not just defend them.
- Mortar teams, released once a base's counter-attacks are spent.
- Civilians in towns the round has reached.
- Server-side save, on the game's own persistence.

NOT IN YET: road and air patrols, Everon map & Kolguyev map 

Mission making, configuration and troubleshooting:
https://github.com/Michael4170/PVEF-Core/blob/main/Mission-Making.md

Suggestions and bug reports welcome via https://discord.gg/SsM7r8b7ae or the GitHub
page.
