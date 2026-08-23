# Administrator Skill Gain
Operation `0x67` raises an office [administrator's](../auto-traders/administrators.md)
trading skill by one displayed level.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x67`|
|0x04|u32|auto-trader index|
|0x08|u32|merchant index|
|0x0c|u32|town index|
|0x10|u32|the new trade skill byte|

The handler at `0x0053DDF0` (operation switch case `0x005364B6`) resolves the office through
`0x005308A0(merchant, town)`, checks that `office+0x2F2` still names the trader in the
payload, writes the byte to the record's trade skill, recomputes the wage (`0x004FE160`) and
posts a type `0x7C` event through `0x004D6530`.

Note that the new skill value is computed by the **producer**, not here: unlike a captain's
[skill gain](./0012-auto-trader-skill-gain.md) there is no ceiling table on this path at
all. Where the 43-point steps come from, and why every administrator eventually reaches
level 5, is on [Administrators](../auto-traders/administrators.md#he-gains-on-a-path-of-his-own).
