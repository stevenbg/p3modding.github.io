# Trading Office
A trading office starts with a complete [storage](../reference/storage.md) struct - the same layout
towns use, so an office's stock of a ware is `office + 0x4 + ware*4` - and continues with
office-specific fields after it. The whole record is `0x44C` bytes.

|Offset|Type|Meaning|
|-|-|-|
|`0x000`|storage|the office's own [storage](../reference/storage.md), stock at `+0x4 + ware*4`|
|`0x2C4`|u16|owning merchant index|
|`0x2C6`|u8|town index|
|`0x2C8`|u16|next office of the same merchant|
|`0x2CA`|u16|next office in the same town|
|`0x2CC`|u16|head of this office's chain of the merchant's **production records**, linked through each record's `+0x8`. They live in a world-wide array whose live slot count is `[0x006DE4A6]` (128 and 256 observed in two saves), resolved by `0x005303B0(game_world, index)` as `[0x006DE510] + index * 0x14`. The **stride is `0x14`**; only the first 16 bytes mirror a [facility](../reference/facilities.md). One record aggregates every building of one type this merchant owns in this town: `+0x0` efficiency, `+0x4` employees, `+0x6` facility type, `+0x7` town index, `+0xC` total worker capacity (the field the building count divides - `+0xA` is a near-always-equal copy of it). See [Merchant Buildings](../towns/production.md#merchant-buildings). These employee counts are one of the two inputs to the [population target](../towns/population.md#population-is-four-times-the-number-of-jobs) - the town's own facilities are the other - and each job there counts for four people|
|`0x2CE`|u16|head of a **different** chain: the merchant's finished business structures as they stand on the town map, one entry per physical building, linked through each record's `+0x4`. These are [construction site](../towns/construction.md) records, 8 bytes, and `0x005200D4` links a site into this chain when it completes. Where `+0x2CC` aggregates production per type, this chain holds the individual buildings|
|`0x2D0`|u16|the same, for the merchant's finished dwellings - the nine dwelling building ids, linked at `0x005200F0`|
|`0x2D2`|u16|business buildings the merchant owns in this town - one per building, verified across several saves and offices. Counted in a loop over the town's buildings (`0x004FFDD9`, `0x004FFE5A`) and added to the administrator's [wage](../auto-traders.md#wages) wherever the interface shows it|
|`0x2D6`|u16|state flags; bit `0x1` means the office holds administrator orders|
|`0x2DE`, `0x2E0`, `0x2E2`|u16 x3|residents of the merchant's houses in this town - rich, wealthy, poor. The house info panel's "All dwellings in this town" adds every office's three words to the town's own (`town + 0x778`..`+0x77C`) for the occupants figure (`0x005B02AE`..`0x005B02D9`)|
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
with them is in [Auto Traders](../auto-traders/administrators.md#the-administrators-trading).

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
