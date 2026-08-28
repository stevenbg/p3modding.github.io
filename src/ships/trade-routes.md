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

## Is the Route Running
**Bit `0` of `ship+0x136`** is the switch: the ships tick only executes a stop for a ship
that has it set, and it is the [ship panel](../ui/ship-panel.md)'s "active" checkbox.

[Operation `0x68`](../operations/0068-set-trade-route-active.md) is what moves it.
Activating writes the whole byte as `0x01` (`0x0053E0F3`); deactivating clears the low
two bits with `and 0xFC` (`0x0053DF4D`). Both paths also set `0x8` in `ship+0x3D`
(`0x0053E0E8` and `0x0053DF49`). A finished stop whose action byte has `0x02` clears
the low bits too, as noted above.

So the byte is not a set of independent flags to be OR-ed at will - the activate path
overwrites it - and the reliable read for "is this ship trading" is `ship+0x136 & 1`.

## A Closed Destination Port
Before a ship at sea reaches its destination, the status `0x0F` handler checks whether
that port will admit it (`0x00506D8E`). It reads the flags of the town in
`field_39_last_town` - the destination while a ship is under way - and refuses entry
when:

- the port is [frozen](../towns/port-freezing.md) (`0x04000000`) or blockaded (`0x200`); or
- flag `0x400` is set **and** the ship belongs to a real merchant with `ship+0x136 & 1`.

A refused ship is turned away rather than stopped mid-ocean (`0x00506E85`): the town that
refused it is recorded in `ship+0x37`, the destination `field_39` is cleared to `0xFF`,
the status stays `0x0F`, and then an AI-owned ship is given a new destination
(`0x0051A2C0`) while a player-owned one goes to `0x004D5900`, which advances
`ship+0x132` to the next stop of its route. Otherwise the handler falls through to
`0x00506EF9`, which splices the ship out of the at-sea list and docks it.

**A ship whose remaining destinations are all closed therefore runs out of stops**, and
the game says so: *"%s's trade route: no destination specified"*. The route status
messages live in a pointer table based at `0x006B04F4`, indexed at `0x005486FD` by a
fall-through switch that formats the ship's inline name (`ship+0x160`) into the `%s`.
Its neighbours in the table are *"%s's trade route is interrupted"*, *"ship condition too
poor"*, *"new ship joined"* and *"crew number too low"*.

This is what makes a winter freeze visible to the player as auto-trade ships going idle:
on a short route whose towns all ice over, there is nothing left to sail to.
