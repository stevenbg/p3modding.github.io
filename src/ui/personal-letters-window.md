# Personal Letters Window
The letters window (envelope button; "Personal letters", "Trade", etc. tabs)
lists the player's [messages](../letters.md). Its object pointer is held in the
static `0x006CBD90`.

|Field|Meaning|
|-|-|
|`+0xA0`|pointer to the row model (heap; freed and rebuilt on tab switches)|
|`+0xCC`|row count (u16)|
|`+0xD2`|selected row (u16), `0xFFFF` = none|
|`+0xD4`|current tab (u16), clamped to `0..3`|

## Row Model
The row model is an array of 20-byte rows built by `0x0047C520`, which walks the
player's mailbox chain and keeps the messages whose category matches the current
tab (category = byte table `0x006C0198` indexed by message type). `0x0047C820`
(thiscall(this, tab, force)) switches tabs and rebuilds. Row layout:

|Offset|Meaning|
|-|-|
|`+0x0`|message pool index (u16); `0xFFFF` terminates the array|
|`+0x2`|message type|
|`+0x4`|title id: a copy of the message's town byte|
|`+0x6`|unread flag (bit 7 of the message's `+0x1`)|
|`+0x8`|the date as text, `dd.mm.yyyy`|

## Row Draw
Each row draws the date string, the type name, and the town column. The town
column is looked up as `[0x006DDA00 + 4*title_id]` (`0x0047D928`) - the town-name
[name bank](./name-banks.md) - **without a bounds check**, and the resulting
pointer goes to the render DLL's text draw, which dereferences it unguarded (see
[Render Imports](../ui.md#render-imports)). Messages whose town byte is not a
town index make this read past the bank into unrelated globals: the row then
shows a wrong town, shows nothing, or crashes the game, depending on the value it
hits - the [patrol letter crash](../bugs/patrol-letter-crash.md).

The unread flag selects the row color: black (`0xFF000000`) for unread, brown
(`0xFF5A2406`) for read. The middle column is the message's type name from the
string table at `0x006A52D0` (indexed by type); the scripted letter types
`0x3C..0x40` show their letter-text payload (`+0xC`) instead.
