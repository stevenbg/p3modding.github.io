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
The file *is* the blob the task's `+0x8` points at:

|Field|Meaning|
|-|-|
|`+0x0`|command count (u16)|
|`+0x2`|variable count (u16)|
|`+0x4`|one u32 offset per command, so `offsets[0] == 4 + count * 4`|
|after them|the command bytes, then the string pool|

Because every command's start is in the offset table, command **lengths** are known
without knowing the commands - a good check when decoding operand widths.

The create-letter command's trailing dword is an offset into the string pool, pointing at
a group of NUL-terminated strings: title, body, then one string per answer button
(`Race`, `Dear Sir or Madam...`, `Participate`, `Refuse`).

## Reading the Commands
A command byte dispatches through the index table at `0x004F3054` (`opcode - 1`) into the
handler table at `0x004F2E34`; valid opcodes are `0x01..0x74` and `0xE4..0xFB`, and
`0x00`, `0xFD`, `0xFE`, `0xFF` are handled before the table at `0x004F2CCB`. Handlers
return through `0x004F2C93`, which advances the program counter, or `0x004F2C97`, which
does not - branches set it themselves. Time is the game clock at `game_world+0x14`, 256
ticks to a day.

|Command|Bytes|Meaning|Handler|
|-|-|-|-|
|`00 v`|2|sleep until the absolute time in `var[v]`; if that is more than 256 ticks past, end the script|`0x004F2D71`|
|`05 dst`|2|`var[dst] = now`|`0x004EEC39`|
|`06 src dst`|3|`var[dst] = now + var[src] * 256` - a date so many days out|`0x004EECE2`|
|`09 mod dst`|3|`var[dst] = pseudo-random % var[mod]`|`0x004EEDFD`|
|`0A dst imm32`|6|`var[dst] = imm32`|`0x004EEEAA`|
|`0B a b dst`|4|add|`0x004EEED1`|
|`0C a b dst`|4|subtract|`0x004EEEF3`|
|`0D a b dst`|4|multiply|`0x004EEF15`|
|`0E a b dst`|4|divide|`0x004EEF36`|
|`17..1C a b T F`|7|compare `var[a]` with `var[b]` and jump to command `T` if it holds, `F` if not: `17` `<`, `18` `<=`, `19` `==`, `1A` `!=`, `1B` `>=`, `1C` `>`|`0x004EF2DF`+|
|`1D T`|3|jump to command `T`|`0x004EF3DF`|
|`1E ship dst`|3|`var[dst]` = the town that ship is in (`ship+0x39`), or `-1` unless its status is below `0xF` and not 2 or 3|`0x004EF3F0`|
|`21 dst`|2|a random town index|`0x004EF4B6`|
|`23 a dst`|3|a random first-name index (bit 7 of the low byte set - a table flag); printed by `%V`|`0x004EF588`|
|`24 dst`|2|a random surname index; printed by `%N`|`0x004EF65F`|
|`28 ship dst`|3|`var[dst]` = the ship's owner (`ship+0x0`)|`0x004EF9DF`|
|`29 a b c d e`|6|route/town picker through `0x00533210`|`0x004EFBAA`|
|`2A town ship dst`|4|`var[dst]` = whether that ship is in that town|`0x004EFDDF`|
|`39 dst`|2|`var[dst]` = the number of merchants whose mailbox gate (`+0x8`) is zero, i.e. human players|`0x004F0F6A`|
|`3A ship flag`|3|`var[flag]` nonzero reserves the ship (`ship+0x137`, capped at `0xFA`), zero releases it|`0x004F0FCF`|
|`43 dst`|2|`var[dst]` = the town count (`game_world+0x10`)|`0x004F1592`|
|`56 dst`|2|`var[dst]` = `[0x006DE52E]`, the home town of the merchant at `[0x006DFC14]` - the Hanse council's seat, set at `0x0041B5D0`|`0x004F1F14`|
|`57 merchant town dst`|4|`var[dst]` = that merchant's best ship docked in that town, `-1` if none|`0x004F01AA`|
|`58 a dst`|3|indirect load, `var[dst] = var[var[a]]`|`0x004F1F2C`|
|`59 idx src`|3|indirect store, `var[var[idx]] = var[src]`|`0x004F1F47`|
|`61 merchant delta`|3|adds `var[delta]` to the merchant's saturating word at `+0x16` through `0x004F36E0`|`0x004F2512`|
|`F1 ship amount dst`|4|cargo: a positive `var[amount]` is loaded onto the ship (`0x00518640`, `var[dst]` = how much fitted), a negative one taken off (`0x005186D0`)|`0x004EE1CD`|
|`F7 ...`|12|[create letter](./scripted-letters.md)|`0x004ED4A0`|
|`F8 merchant amount`|3|**pay**: `merchant+0x0 += var[amount]`, booked to `+0x4B8` (income) or `+0x4BC` (expenses) by sign|`0x004ED20D`|
|`FD v T`|4|sleep `var[v]` ticks, resume at command `T`|`0x004F2CF8`|
|`FE v`|2|sleep `var[v]` days and restart at command 0|`0x004F2DB1`|
|`FF`|1|end the script|`0x004F2DEF`|

`F8` is what makes a variable money: a mission's "reward" is whatever variable reaches
this command, and a negative amount is a charge - the treasure map's asking price and the
race's entry stake are both taken this way.

## The Letter Templates
Two conventions in the string pool make the templates worth reading before the bytecode.

A `%` placeholder is followed by a code letter and then a **variable index**, so the text
names the variables it prints: `%t` a town, `%c` a sum of money, `%s` a ship, `%B` an
amount with the loads symbol, `%D` a date, `%N`/`%V`/`%R`/`%v` name parts, `%+` the
signature block. An index of `0` puts a NUL *inside* the template, so a parser has to
consume the argument byte rather than treat it as the end of the string.

A `|` separates the question the offer asks from the reply the player gets once he
accepts. That is what makes a smuggler's destination a secret and a trader's public: both
scripts name the destination in their letter, but the smuggler only after the bar.

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
