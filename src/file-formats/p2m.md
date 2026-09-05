# Mission Scripts (.p2m)

A `.p2m` file is a compiled **letter script**: the bytecode the interpreter at
`0x004ECF64` runs for missions and scripted letters. What the scripts *do*, which ones
exist and how the game loads them is on [Mission Scripts](../letters/mission-scripts.md);
this page is the file format. `p2mcli` in the p3-lib repository disassembles and lints
them; the 94 shipped scripts live under `missions_addon/` in `p2arch0_eng.cpr` and
`p2arch1_eng.cpr` (see [CPR](./cpr.md)).

## Layout

|Field|Meaning|
|-|-|
|`+0x0`|command count (u16)|
|`+0x2`|variable count (u16)|
|`+0x4`|one u32 file offset per command, so `offsets[0] == 4 + count * 4`|
|after them|the commands, back to back|
|after the last command|the string pool: NUL-terminated latin1 strings|

Every command's start is in the offset table, so a command's **length** is known without
knowing the command - which is how the lengths below were checked: across the 94 shipped
scripts, no opcode ever appears with two different lengths.

A command is an opcode byte followed by its operands. Operands are one byte each and
name a **variable** by index (`var[n]`, a dword), except: the jump targets of `17`..`1D`
and `FD` are u16 **command indices**, `0A` carries a dword immediate, and `F7` ends in a
dword **string-pool offset**. Variables hold whatever the last writer put there - a time,
a town, a ship, a sum of money - and a text placeholder prints one by index.

## The string pool

The create-letter command's pool offset points at a group of NUL-terminated strings:
title, body, then one string per answer button. Two conventions inside them:

- A `%` placeholder is followed by a code letter and then a **variable index byte**:
  `%t` a town, `%c` a sum of money, `%s` a ship, `%B` an amount with the loads symbol,
  `%D` a date, `%N`/`%V`/`%R`/`%v` name parts, `%+` the signature block. An index of `0`
  puts a NUL *inside* the string, so a parser has to consume the argument byte rather than
  treat it as the terminator.
- A `|` separates the question the offer asks from the reply the player gets once he
  accepts.

## Opcodes

A command byte dispatches through the index table at `0x004F3054` (`opcode - 1`) into
the handler table at `0x004F2E34`; valid opcodes are `0x01..0x74` and `0xE4..0xFB`, and
`00`, `FD`, `FE`, `FF` are handled before the table at `0x004F2CCB`. Handlers return
through `0x004F2C93`, which advances the program counter, or `0x004F2C97`, which does
not - branches set it themselves. Three opcodes share a handler with another (`26`/`27`,
`30`/`3D`, `E7`/`E9`/`F9`).

The table lists every valid opcode: its length (from the shipped scripts, `-` where no
script uses it), how many times the shipped scripts use it, its handler, and its meaning
where that has been traced. 100 of the 144 are not traced yet, and 21
are used by no shipped script.

|Op|Len|Uses|Handler|Meaning|
|-|-|-|-|-|
|`00`|2|49|`0x004F2D71`|`00 v`: sleep until the absolute time in `var[v]`; if that is more than 256 ticks past, end the script|
|`01`|3|10|`0x004EEA22`||
|`02`|-|0|`0x004EEA6E`||
|`03`|4|1|`0x004EEAAB`||
|`04`|2|23|`0x004EEB62`||
|`05`|2|84|`0x004EEC39`|`05 dst`: `var[dst] = now`|
|`06`|3|56|`0x004EECE2`|`06 src dst`: `var[dst] = now + var[src] * 256` - a date so many days out|
|`07`|3|20|`0x004EED2D`|`07 merchant dst`: `var[dst]` = that merchant's home town|
|`08`|2|1|`0x004EED7B`||
|`09`|3|84|`0x004EEDFD`|`09 mod dst`: `var[dst] = pseudo-random % var[mod]`|
|`0A`|6|1487|`0x004EEEAA`|`0A dst imm32`: `var[dst] = imm32`|
|`0B`|4|367|`0x004EEED1`|`0B a b dst`: add|
|`0C`|4|105|`0x004EEEF3`|`0C a b dst`: subtract|
|`0D`|4|58|`0x004EEF15`|`0D a b dst`: multiply|
|`0E`|4|39|`0x004EEF36`|`0E a b dst`: divide|
|`0F`|-|0|`0x004EEF59`||
|`10`|-|0|`0x004EEF8A`||
|`11`|3|12|`0x004EF112`||
|`12`|3|5|`0x004EF161`||
|`13`|-|0|`0x004EF1AF`||
|`14`|-|0|`0x004EF1FD`||
|`15`|3|31|`0x004EF260`||
|`16`|-|0|`0x004EF2B6`||
|`17`|7|146|`0x004EF2DF`|`17 a b T F`: if `var[a] < var[b]` jump to command `T`, else `F`|
|`18`|7|9|`0x004EF30B`|`18 a b T F`: `<=`, same shape|
|`19`|7|51|`0x004EF337`|`19 a b T F`: `==`|
|`1A`|7|50|`0x004EF35F`|`1A a b T F`: `!=`|
|`1B`|7|108|`0x004EF387`|`1B a b T F`: `>=`|
|`1C`|7|42|`0x004EF3B3`|`1C a b T F`: `>`|
|`1D`|3|79|`0x004EF3DF`|`1D T`: jump to command `T`|
|`1E`|3|13|`0x004EF3F0`|`1E ship dst`: `var[dst]` = the town that ship is in (`ship+0x39`), or `-1` unless its status is below `0xF` and not 2 or 3|
|`1F`|4|5|`0x004EF788`||
|`20`|2|1|`0x004EF485`||
|`21`|2|17|`0x004EF4B6`|`21 dst`: a random town index|
|`22`|2|3|`0x004EF518`||
|`23`|3|7|`0x004EF588`|`23 a dst`: a random first-name index (bit 7 of the low byte set - a table flag); printed by `%V`|
|`24`|2|13|`0x004EF65F`|`24 dst`: a random surname index; printed by `%N`|
|`25`|-|0|`0x004EF70C`||
|`26`|4|1|`0x004EF875`||
|`27`|-|0|`0x004EF875`||
|`28`|3|15|`0x004EF9DF`|`28 ship dst`: `var[dst]` = the ship's owner (`ship+0x0`)|
|`29`|6|3|`0x004EFBAA`|`29 a b c d e`: route/town picker through `0x00533210`; writes towns to `a`..`d`|
|`2A`|4|6|`0x004EFDDF`|`2A town ship dst`: `var[dst]` = whether that ship is in that town|
|`2B`|5|5|`0x004F028D`||
|`2C`|4|4|`0x004F0692`||
|`2D`|3|1|`0x004F06C1`||
|`2E`|4|10|`0x004F0711`||
|`2F`|4|1|`0x004F0780`||
|`30`|-|0|`0x004F2D44`||
|`31`|3|1|`0x004F0816`||
|`32`|-|0|`0x004F086A`||
|`33`|3|1|`0x004F098E`||
|`34`|3|16|`0x004F0A27`|`34 town dst`: `var[dst]` = the town's citizens (`town+0x2D4`)|
|`35`|2|1|`0x004F0A54`||
|`36`|4|2|`0x004F0A6F`||
|`37`|9|48|`0x004F0E94`||
|`38`|4|12|`0x004F0EFA`||
|`39`|2|17|`0x004F0F6A`|`39 dst`: `var[dst]` = the number of merchants whose mailbox gate (`+0x8`) is zero, i.e. human players|
|`3A`|3|29|`0x004F0FCF`|`3A ship flag`: `var[flag]` nonzero reserves the ship (`ship+0x137`, capped at `0xFA`), zero releases it|
|`3B`|3|8|`0x004F1001`||
|`3C`|-|0|`0x004F1050`||
|`3D`|-|0|`0x004F2D44`||
|`3E`|4|1|`0x004F11A5`||
|`3F`|3|17|`0x004F1253`||
|`40`|6|1|`0x004F12BB`||
|`41`|4|1|`0x004F1409`||
|`42`|3|7|`0x004F1514`||
|`43`|2|11|`0x004F1592`|`43 dst`: `var[dst]` = the town count (`game_world+0x10`)|
|`44`|3|6|`0x004F15AC`|`44 merchant dst`: `var[dst]` = a random town the merchant has an office in|
|`45`|3|1|`0x004F167C`||
|`46`|3|3|`0x004F16B3`||
|`47`|5|4|`0x004F17C0`||
|`48`|5|4|`0x004F188D`||
|`49`|4|6|`0x004F192D`||
|`4A`|-|0|`0x004F1987`||
|`4B`|5|1|`0x004F1A0F`||
|`4C`|5|6|`0x004EE30B`||
|`4D`|4|2|`0x004F1A96`||
|`4E`|3|1|`0x004F1B13`||
|`4F`|4|3|`0x004F1B74`||
|`50`|6|3|`0x004F1C74`||
|`51`|2|4|`0x004F1DCA`||
|`52`|-|0|`0x004F1E38`||
|`53`|4|2|`0x004EFB40`||
|`54`|4|1|`0x004EFA54`||
|`55`|-|0|`0x004EFE91`||
|`56`|2|9|`0x004F1F14`|`56 dst`: `var[dst]` = `[0x006DE52E]`, the home town of the merchant at `[0x006DFC14]` - the Hanse council's seat|
|`57`|4|2|`0x004F01AA`|`57 merchant town dst`: `var[dst]` = that merchant's best ship docked in that town, `-1` if none|
|`58`|3|18|`0x004F1F2C`|`58 a dst`: indirect load, `var[dst] = var[var[a]]`|
|`59`|3|6|`0x004F1F47`|`59 idx src`: indirect store, `var[var[idx]] = var[src]`|
|`5A`|5|1|`0x004F00AC`||
|`5B`|3|2|`0x004F1F62`||
|`5C`|5|1|`0x004F1FB3`||
|`5D`|4|1|`0x004F20E6`||
|`5E`|5|2|`0x004F212B`||
|`5F`|7|1|`0x004F2216`||
|`60`|3|3|`0x004F24E3`||
|`61`|3|2|`0x004F2512`|`61 merchant delta`: adds `var[delta]` to the merchant's saturating word at `+0x16` through `0x004F36E0`|
|`62`|4|3|`0x004F2565`||
|`63`|5|1|`0x004F2363`||
|`64`|2|7|`0x004F25E8`|`64 pct`: sets `ship+0x136` bit `0x20` on that share of every AI merchant's ships|
|`65`|3|25|`0x004F26C0`|`65 id dst`: `var[dst]` = the town index for a town id, `-1` when out of range|
|`66`|3|3|`0x004F272E`||
|`67`|6|1|`0x004F27BE`||
|`68`|9|1|`0x004F286E`||
|`69`|4|1|`0x004F2A89`||
|`6A`|4|3|`0x004F2B3A`||
|`6B`|3|1|`0x004ED332`||
|`6C`|3|2|`0x004F2BDA`||
|`6D`|5|1|`0x004EFF63`||
|`6E`|6|3|`0x004F03AC`||
|`6F`|7|3|`0x004EFCB9`|`6F a b c d ...`: town picker; writes towns to `a`..`d`|
|`70`|4|1|`0x004F0C74`||
|`71`|-|0|`0x004ED018`||
|`72`|3|1|`0x004EF062`||
|`73`|-|0|`0x004EEC65`||
|`74`|-|0|`0x004F051A`||
|`E4`|-|0|`0x004ED00A`||
|`E5`|3|1|`0x004EE698`||
|`E6`|3|8|`0x004F2C3E`||
|`E7`|5|1|`0x004ED125`||
|`E8`|2|1|`0x004ED086`||
|`E9`|5|2|`0x004ED125`||
|`EA`|-|0|`0x004EE9DB`||
|`EB`|3|2|`0x004EE984`||
|`EC`|2|10|`0x004EE7D0`|`EC town`: start a plague there - operation `0x7C`, scheduled task `0x1C`; refused if the town already has flag `0x8`|
|`ED`|2|3|`0x004EE567`||
|`EE`|6|11|`0x004EE44A`||
|`EF`|2|14|`0x004EE25C`||
|`F0`|3|3|`0x004EE238`||
|`F1`|4|6|`0x004EE1CD`|`F1 ship amount dst`: cargo - a positive `var[amount]` is loaded onto the ship (`0x00518640`, `var[dst]` = how much fitted), a negative one taken off (`0x005186D0`)|
|`F2`|3|11|`0x004EE1A2`||
|`F3`|3|4|`0x004EE155`||
|`F4`|4|3|`0x004EE08A`||
|`F5`|-|0|`0x004EE012`||
|`F6`|6|4|`0x004EDF3F`||
|`F7`|12|203|`0x004ED4A0`|`F7 kind town d1..d5 text32`: [create letter](../letters/scripted-letters.md) - letter type `0x3C + kind`, town variable, five descriptor bytes, then a dword offset into the string pool|
|`F8`|3|37|`0x004ED20D`|`F8 merchant amount`: **pay** - `merchant+0x0 += var[amount]`, booked to `+0x4B8` (income) or `+0x4BC` (expenses) by sign|
|`F9`|5|4|`0x004ED125`||
|`FA`|2|1|`0x004ED3A6`||
|`FB`|3|1|`0x004ED2A7`||
|`FD`|4|231|`0x004F2CF8`|`FD v T`: sleep `var[v]` ticks, resume at command `T`|
|`FE`|2|35|`0x004F2DB1`|`FE v`: sleep `var[v]` days and restart at command 0|
|`FF`|1|77|`0x004F2DEF`|`FF`: end the script|
