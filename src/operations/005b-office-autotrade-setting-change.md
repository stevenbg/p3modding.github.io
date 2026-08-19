# Office Autotrade Setting Change
Operation `0x5B` sets one ware's administrator trade order in a trading office: the
stock amount and the price, where the price's sign encodes the direction. It is
enqueued by the trading office window's administrator view ("Trading Office" side
button).

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x5B`|
|0x04|i32|stock amount, raw units|
|0x08|i32|price: positive = sell (minimum price), negative = buy (maximum price, negated), 0 = no order|
|0x0C|u32|office index|
|0x10|u32|ware index|

The handler at `0x0053D5C0` (operation switch case `0x536290`) resolves the office by
index and writes the stock to `office+0x354+ware*4` and the price to
`office+0x2F4+ware*4`. It also records the price into the owning merchant's per-ware
price memory and maintains the office's "has administrator orders" flags at
`office+0x2D6`, clearing them when every price is 0.

If the office's administrator index (`office+0x2F2`) is invalid, the handler instead
clears all 24 prices and stock amounts - an office without an administrator cannot
hold orders.

There is no direction field: the administrator view's direction arrows are purely a
rendering of the price's sign.
