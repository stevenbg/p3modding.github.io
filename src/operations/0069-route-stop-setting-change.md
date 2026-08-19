# Route Stop Setting Change
Operation `0x69` changes one ware's instruction of an applied trade route stop: the
amount and the price, in the stop record encoding of the [.rou format](../file-formats/rou.md).
It is enqueued by the "Automatic maritime trading" dialog (the route window's Goods
button): its +/- buttons keep edits pending in the dialog object and commit them
through this operation when the edited ware changes, the stop is switched, or the
dialog closes - which is also why the dialog's Undo only covers edits since the last
such commit.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x69`|
|0x04|u16|route stop pool index|
|0x06|u16|merchant index|
|0x08|u16|ware index|
|0x0A|u16|instruction slot in the stop's ware order array|
|0x0C|i32|amount, raw units; negative = ship to office; `1_000_000_000` = Max|
|0x10|i32|price: positive = sell minimum, negative = buy maximum, 0 = office transfer|

The handler at `0x0053E480` (operation switch case `0x5364D4`) writes the values into
the stop's record in the route stop pool at `[0x006DD72C]`.

The dialog also uses this operation to normalize a stop's inactive slots after opening:
slots without an instruction (amount 0) can carry leftover base prices, and the dialog
enqueues one operation per such ware to zero them, drained over the following ticks.
Code reading a stop record must therefore treat `amount != 0` as the "slot has an
instruction" test - the price alone can be a stale base price for a while.
