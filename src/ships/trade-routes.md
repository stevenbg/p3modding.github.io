# Trade Routes
A ship on automatic trade works through a chain of **route stops** - the runtime form of a
[`.rou` file](../file-formats/rou.md#applied-routes-at-runtime). This page is what happens
when the ship reaches one.

## Running a Route Stop
The executor is `0x004D5200` (thiscall on `0x006DD728`, arguments `(ship, office)`). It runs
from the **ships tick** (`0x00506720`), not from an operation: the tick tests bit 0 of
`ship+0x136`, and `0x00518860` resolves the office for the ship's merchant and current town
through the office lookup `0x005308A0` before calling the executor on the ship's current
stop.

The stop's instructions are executed in **two passes over the whole ware order array**,
never interleaved:

|Pass|Handles|
|-|-|
|1|selling to the town (positive price) and unloading into the office (zero price, negative amount)|
|2|buying from the town (negative price) and loading from the office (zero price, positive amount)|

Between them, `0x004D5600` calls the ship's recompute (`0x005182B0`, see
[Crew, Cutlasses and the Equipment Weight](../ships.md#crew-cutlasses-and-the-equipment-weight))
and keeps the returned **free capacity** as a budget, clamped at zero. Pass 2 caps every
purchase and every office load against that budget and decrements it as it spends.

Two consequences worth knowing:

- **Unloading and selling always happen before loading and buying**, for every ware, no
  matter how the order array is arranged - the ship frees space first and the capacity
  budget is measured afterwards. Confirmed in-game.
- The order array only sequences wares **within** a pass. It still matters there: the
  capacity budget and the merchant's cash are consumed in that sequence during pass 2, so
  earlier entries get first claim on the hold when not everything fits.

Each pass walks all 24 slots of the order array; an entry outside `0..0x17` is **skipped**
rather than ending the walk, and a ware whose amount is `0` has no instruction. Every
quantity is floored to a whole in-game unit (the barrel/bundle scaling table at
`0x00672C14`) before anything moves. The amount field is a cap, not a target:

|Instruction|Quantity|
|-|-|
|unload into the office|`-amount`, capped by what is aboard|
|load from the office|`amount`, capped by [`0x00500EC0`](../merchants/trading-office.md#what-the-lock-bit-does) and by the capacity budget|
|sell to the town|`0x0052EA80(ware, town, price) - town stock`, capped by `amount`|
|buy from the town|capped by `amount`, by the capacity budget, and by the merchant's cash|

A purchase the merchant cannot fully afford is **scaled down proportionally**
(`0x004D5705`) rather than skipped, and its cost runs through the captain's
[buying discount](../auto-traders/skill.md#the-buying-discount). Goods entering the office
go through [`0x004FF6F0`](../merchants/trading-office.md#average-purchase-price), which is
how the office's average purchase price follows the cargo.

Finally, the stop's [action byte](../file-formats/rou.md#action-byte) is consulted twice:
bit `0x04` (the first stop of the route) builds a record from the captain's name ids and the
ship's registry id and passes it to `0x004D6530`, and bit `0x02` clears the low bits of
`ship+0x136` once the stop is finished.
