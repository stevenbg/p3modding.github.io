# Buy Militia Weapons

Operation `0x50` moves militia weapons between the **tavern's weapons dealer** and one
of the player's ships or his trading office. Handler `0x0053BC90` (operation switch
case `0x005360DE`); sent from the tavern window at `0x005D661C` and `0x005D66DE` - the
same window class that sends the burglar hire
([operation `0x4F`](../scheduled-tasks/0005-criminal-investigation.md#how-investigations-start)).

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x50`|
|0x08|i32|amount - the operation is rejected unless it is positive|
|0x0C|u8|town index|
|0x0D|u8|merchant index|
|0x0E|u8|weapon selector, masked to its low 2 bits|
|0x0F|u8|direction flag; the tavern sends `1`|
|0x10|u16|ship index, or out of range to use the trading office instead|

The traded ware is `(field_E & 3) + 0x14`, i.e. one of
[`Sword`, `Bow`, `Crossbow`, `Carbine`](../reference/wares.md) - the militia weapons,
not [ship artillery](../reference/ship-artillery.md).

With a valid ship index the goods go to the ship's ware array (`ship + 0x54 + ware*4`,
mirrored at `+0xB4`); the free capacity is recomputed with `0x005182B0` and the amount
is clamped to a tenth of it. Otherwise the office for that merchant and town
(`0x005308A0`) receives them at `office + 0x4 + ware*4`.

Dealing with the weapons dealer is criminal business: right after the validity checks,
`0x0053BD00` adds the constant at `0x00672DDC` - **25** - to the merchant's
[underworld reputation](../merchants/underworld-reputation.md).
