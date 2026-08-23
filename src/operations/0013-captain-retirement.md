# Captain Retirement
Operation `0x13` retires the [auto trader](../auto-traders.md) commanding a ship once he
is old enough, and tells the player about it.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x13`|
|0x04|u32|ship index|
|0x08|u32|auto-trader index|

The handler at `0x00538C40` (operation switch case `0x00535980`) validates both indices,
then:

1. schedules task `0x27` (`0x004DDC00`) for the next tick, carrying the ship index at
   `data+0x0` and the trader index at `data+0x4` - that task is what actually takes the
   captain off the ship;
2. sets `field_E` on the auto-trader record, which stops the
   [ten-day update](../scheduled-tasks/0003-ten-day-update.md) queueing a second removal;
3. for a ship whose owner is a human player (`merchant+0x8` = 0), composes a type `0x52`
   message from the captain's name ids and the ship's registry id and posts it through
   `0x004D6530`.

The operation is only ever enqueued by the ten-day update, and only for a human player's
ships - an AI merchant's captains are retired by that task directly, without an operation
and without a message.
