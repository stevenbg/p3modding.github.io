# Hand Ship to Pirate
Operation `0x0D` marks one of the player's ships as sailing for a
[pirate captain](../pirates.md), or clears that mark again.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x0D`|
|0x04|u32|ship index|
|0x08|u32|non-zero to raise the black flag, `0` to clear it|

The handler at `0x005386C0` (operation switch case `0x005358D5`) takes one of two paths.
For a ship not in a convoy it requires status `0x0F` - a merchant vessel at sea - and then
writes only three fields: `field_15C_is_pirate = 1`, `field_3D |= 0x18` and
`field_136 = 0`. Clearing instead writes `field_15C_is_pirate = 0` and `field_3D |= 0x10`.
For a ship in a convoy it ORs `0x1000` into the convoy's `+0x18` and walks the convoy's
ships.

Note what it does **not** do: the ship keeps its owner and its status. Both change later,
when the ship actually puts to sea as a raider - see
[Bands and Hideouts](../pirates.md#bands-and-hideouts).
