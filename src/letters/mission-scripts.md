# Mission Scripts

Missions are not compiled into the executable. Each one is a **file** - a blob of the
same [letter script](./scripted-letters.md) bytecode the interpreter at `0x004ECF64`
runs - shipped inside the game's archives, so a mission's whole behaviour can be read
statically.

## Where They Live
`scripts/missions_eng.ini` (in `p2arch0_eng.cpr`) maps script ids to file names in its
`[Missions]` section, and the `[GroupX]` sections decide which ids a game mode loads:
`[EinzelSpiel]` (single player) takes `History Standard`, while `[Multiplay]` and
`[Hotseat]` add `Standard2 Multiplay`. The loader at `0x005116F0` reads
`[Missions] <id>`, prefixes `missions_addon/` (`0x006C04AC`) and opens the file.

The tavern's [side-room missions](./71-tavern-missions.md) are these ids:

|Id|File|Mission|
|-|-|-|
|8|`Schmuggler.p2m`|smuggler|
|9|`TransportAuftrag.p2m`|trader (transport order)|
|11|`PiratVernichten.p2m`|pirate hunter|
|12|`SchatzKarte.p2m`|treasure map|
|13|`Reisender.p2m`|courier|
|15|`Eskorte.p2m`|escort and fugitive (one script, two letters)|
|16|`patrouille.p2m`|patrol|

`p2arch0_eng.cpr` holds 93 such files under `missions_addon/`; `p2arch1_eng.cpr` is a
patch archive containing a single replacement, `missions_addon/WettbewerbRennen.p2m`.

## File Layout
The file *is* the blob the task's `+0x8` points at: a command count, a variable count, an
offset table, the commands and the string pool - the format, with every opcode and its
length, is on [Mission Scripts (.p2m)](../file-formats/p2m.md).

## Reading the Commands
A command byte dispatches through the index table at `0x004F3054` (`opcode - 1`) into the
handler table at `0x004F2E34`; the [opcode table](../file-formats/p2m.md#opcodes) has
every handler. Time is the game clock at `game_world+0x14`, 256 ticks to a day. The
commands a mission is built from: `05`/`06` take the time and a date so many days out,
`0A` loads a constant, `0B`..`0E` do arithmetic, `17`..`1C` compare two variables and jump,
`FD`/`FE`/`00` sleep, `F7` creates the letter, `3A` reserves a ship, and `F8` moves money.

`F8` is what makes a variable money: a mission's "reward" is whatever variable reaches
this command, and a negative amount is a charge - the treasure map's asking price and the
race's entry stake are both taken this way.

The letter templates in the string pool are worth reading before the bytecode: a `%`
placeholder names the variable it prints, and a `|` separates the offer from the reply
after acceptance (the conventions are on the [format page](../file-formats/p2m.md#the-string-pool)).
That is what makes a smuggler's destination a secret and a trader's public: both scripts
name the destination in their letter, but the smuggler only after the bar.

## The Multiplayer Race Never Runs
`WettbewerbRennen.p2m` (id 18, multiplayer and hotseat only) is a complete Hanseatic
League race: a random wait of 200-400 days, an invitation letter with *Participate* and
*Refuse*, a 3,000 stake charged with `F8`, a start town and the town half the map away as
the finish, each entrant's best docked ship reserved with `3A`, an arrival poll, and a
prize of `participants * 3000 + 5000` for the winner.

None of it can execute. Command 11 is `if human_players < 2 -> command 0 else -> command
153`, and command 153 is inside the closing release-and-restart loop rather than command
12 where the invitation begins - every other conditional in the file has its own
fall-through as one of the two targets, this one has neither. Walking the graph from the
entry point reaches 25 of the file's 166 commands; entering at command 12 instead reaches
all 166. Whatever the player count, the script only loops "wait, count players, restart".
