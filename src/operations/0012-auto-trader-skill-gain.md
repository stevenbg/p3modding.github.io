# Auto Trader Skill Gain
Operation `0x12` raises the skills of the [auto trader](../auto-traders.md) commanding a
ship. It is the only writer of the three skill bytes in the game.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x12`|
|0x04|u32|ship index - the record is that ship's `field_42_captain_index`|
|0x08|u32|navigation gain|
|0x0c|u32|trade **and** combat gain - the handler reads this field twice|
|0x10|u32|never read|

The handler at `0x00538A80` (operation switch case `0x00535971`) bails unless the ship index
is below the ship count and the ship carries a captain index below the auto-trader count.
For each of the three skills it adds that skill's gain to the current byte, clamps the
result against the skill's ceiling, and finishes by recomputing the
[wage](../auto-traders.md#wages) from the new skills (`0x004FE190`).

The clamp, the ceilings, the shared trade/combat gain field and what all of that does to a
captain's career are on [Skill](../auto-traders/skill.md). The two producers that enqueue
this operation are the [ten-day world update](../scheduled-tasks/0003-ten-day-update.md) and
a pirate raider reaching its [hideout](../pirates/bands.md).
