# Trading Office
A trading office starts with a complete [storage](./storage.md) struct - the same layout
towns use, so an office's stock of a ware is `office + 0x4 + ware*4` - and continues with
office-specific fields after it. The whole record is `0x44C` bytes.

|Offset|Type|Meaning|
|-|-|-|
|`0x000`|storage|the office's own [storage](./storage.md), stock at `+0x4 + ware*4`|
|`0x2C4`|u16|owning merchant index|
|`0x2C6`|u8|town index|
|`0x2C8`|u16|next office of the same merchant|
|`0x2CA`|u16|next office in the same town|
|`0x2D6`|u16|state flags; bit `0x1` means the office holds administrator orders|
|`0x2F2`|u16|the administrator, as an index into the [auto trader](../auto-traders.md) array; out of range when the office has none|
|`0x2F4`|i32[24]|administrator order price per ware, the sign encoding the direction|
|`0x354`|i32[24]|administrator minimum store quantity per ware, raw units|
|`0x3B4`|u32|"lock min. store quantity" bitmap, one bit per ware|
|`0x3B8`|f32[24]|average purchase price of the stock, per ware|

The order arrays are written by
[operation `0x5B`](../operations/005b-office-autotrade-setting-change.md) and the lock
bitmap by [operation `0x66`](../operations/0066-office-autotrade-lock-change.md). There is
no direction field: a positive price is a sell order's minimum price, a negative one a buy
order's maximum price negated, and `0` means no order. What the administrator then does
with them is in [Auto Traders](../auto-traders.md#the-administrators-trading).

## Average Purchase Price
`0x004FF6F0` (thiscall on the office, arguments `(ware, quantity, price)`) is how goods
enter an office's stock with a price attached. It adds `quantity` raw units to the ware's
stock and folds `price` into the running average at `office + 0x3B8 + ware*4`:

```
average = (average * old_stock + quantity * price) / (old_stock + quantity)
```

A slot that was empty takes the incoming price directly instead. This average is the
purchase price the trading office UI shows for the stock on hand, and it is on the same
per-unit basis as a ship's `field_B4` average prices - which is exactly what the caller
passes when a ship unloads cargo into the office.

## What the Lock Bit Does
`0x00500EC0` (thiscall on the office, argument `(ware)`) answers "how much of this ware
may an auto trader take out of here?":

```
administrator index out of range      -> current stock
lock bit for this ware clear          -> current stock
minimum store quantity <= 0           -> current stock
otherwise                             -> max(current stock - minimum store quantity, 0)
```

So the "Lock min. store quantity for auto trade ships" checkbox does exactly what its
name says, and only in that one direction: it fences the administrator's minimum store
quantity off from ships **loading** at the office. It does not restrain the
administrator's own trading, and it does not affect ships unloading into the office.
