# Office Autotrade Lock Change
Operation `0x66` sets or clears one ware's "Lock min. store quantity for auto trade
ships" checkbox in a trading office's administrator view.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x66`|
|0x04|u32|ware index (validated < 0x18)|
|0x08|u16|merchant index|
|0x0C|u16|town index|
|0x10|u32|lock: 0 clears the bit, anything else sets it|

The handler (operation switch case `0x53644B`; the equivalent standalone handler is
`0x0053DD90`) resolves the office through the office lookup at `0x005308A0`, whose
argument order is (merchant, town), and toggles the ware's bit in the office's lock
bitmap at `office+0x3B4`. On a failed office lookup the operation is silently dropped.

The administrator view draws the checkbox directly from the bitmap (reads at
`0x005D9D98` and `0x005DD987`, passing the player merchant global `operations+0x924`
and the window's town).
