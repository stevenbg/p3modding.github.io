# Auto Traders
Captains, administrators and pirate captains are represented by the same struct. The array
lives behind the ships container at `0x006DD7A0` (array pointer at `+0x0`, count
word at `+0xF2` = `0x006DD892`, stride `0x10`). Each town chains its auto traders
through `field_0_next_auto_trader_index`, headed by the town's
`field_82E_auto_trader_chain_head` (see [Towns](./towns.md)). The chain mixes two
record kinds, discriminated by `field_8` (check `0x004FE150`, true for
`field_8 > 0x20`). The record initializer (`0x004FDF50`) fills `field_8` with a
random byte and reduces it `% 11` for captains (0..10), so the range encodes the
kind: captains always pass `<= 0x20`, the pirate captains - one record per town,
maintained by the spawn task - fail it. (The pirates are the tavern characters a
ship can be handed to; identified by matching the records' name ids against the
pirate captains in-game.) For pirates `field_8` doubles as the greed byte: the
loot share a tavern pirate demands is `25 + 5 * ceil(field_8 / 32)`, i.e.
35%..65% - verified against seven live pirates. The initializer also rolls
`field_2`/`field_3` as first/last
name ids (modulo the name-registry counts `0x006DDB70`/`0x006DDB74`), splits a
600-point budget randomly across the three skills, and stamps `field_4` from the
current date serial. The two kinds also differ in their wage formula: captain
records derive it from the trade skill alone (`0x004FE160`), pirate records from
the sum of all three skills plus a base. A town's tavern offers a captain
for hire while the chain contains a captain record no merchant employs
(`field_F_merchant_index` = `0xFF`): the captain resolver `0x005269A0`(town,
merchant) walks the chain applying exactly that, preferring a captain the asking
merchant already employs; the sibling resolver `0x005261D0` does the same for
the pirate captains. Verified against a live save: exactly the towns whose
taverns showed captains had a matching chain record.
The following fields have been identified:
```c
struct auto_trader
{
  unsigned __int16 field_0_next_auto_trader_index;
  signed __int16 field_2;
  int field_4_timestamp;
  unsigned __int8 field_8;
  unsigned __int8 field_9_navigation_skill;
  unsigned __int8 field_A_trade_skill;
  unsigned __int8 field_B_combat_skill;
  __int16 field_C_daily_wage;
  char field_E;
  unsigned __int8 field_F_merchant_index;
};
```

## Array Layout
Observed live (128-record array): indices 0..47 hold one world-generation pair
per town - the founding captain of an AI merchant's starting ship (elite skills,
exceeding the 600-point budget of the normal roller; verified by matching the
records against the AI ships' `field_42_captain_index` and owners) and the
town's pirate. The dynamic range
above holds spawned tavern captains and employed records; the tail is free
capacity, `memset` to `0xFF` and freelist-linked through `field_0` (the
allocator at `0x005097C0` grows the array by 64 records). Dismissing an office
administrator frees his record back to the freelist; re-employing allocates a
fresh one (new name and skills, wage 10, trade skill 0) - whatever the office
remembers about a previous administrator is stored on the office, not in this
array.

A record's situation is encoded by chain membership, verified across live save
states: chained to a town = sitting in that town's tavern; unchained = serving
on a ship (`field_42_captain_index`). The record's merchant byte is only
maintained for the player - AI merchants' hires keep `0xFF` - so ship ownership
comes from the ship's `field_0_merchant_index` (AI merchants hold low indices
with two starting ships each; pirate ships and empty ship slots hold `0xFF`).
Records never expire; the population circulates between taverns and decks.

## Captain and Pirate Spawning
A periodic task (`0x004E2634`, rescheduling itself in [game ticks](./basics/time.md),
256 per day) maintains both populations. Per town it records two flags: whether
the captain resolver finds an unemployed captain (called **with the merchant
count as the asking merchant** - a value no real merchant has, so only records
with `field_F_merchant_index` = `0xFF` count), and whether the pirate resolver
finds a pirate.

- **Pirates**: when fewer than 3 towns have a pirate, one is spawned into a
  pirate-less town (`0x00526A50(town, 1)` - the pirate initializer path).
- **Captains**: at 8 or more unemployed captains the task just reschedules far
  out (`+0x700` ticks = 7 days). Below that it compares the count against a
  demand target derived from fleet statistics (`0x00509930` on the ships
  container) and spawns a captain into a captain-less town when the count is at
  or below the target, or below 2 (`0x00526A50(town, 0)`), rescheduling `+0x200`
  ticks (2 days) after a spawn and `+0x400` (4 days) otherwise.

No expiry logic exists in the task, and `field_4` is never compared against the
current date: an unhired captain stays until somebody hires him - including AI
ships, which fill their `field_42_captain_index` through the same resolver and
unlink the captain from the town (`0x0051A1B9`). A captain "disappearing" from a
tavern is somebody else's hire, not a timeout.

Dismissing a captain re-links him into the town's chain still carrying the
dismissing merchant's index; it only flips to `0xFF` (generally hireable) when
that town's tavern is next opened (observed in-game). Until then the spawn task
does not count him - so dismissing captains without revisiting their taverns
makes the game under-count and spawn extras, pushing the world above the usual
two hireable captains (four observed live).

## Buying Discount
Auto traders buy cheaper as their trade skill grows. The captain (`0x004D5347`) and
administrator (`0x004FF7E8`) buying routines both compute the percentage of the
transaction price to pay from the auto trader's `field_A_trade_skill`:

```
percent_paid = 2 * (50 - trade_skill / 43)
```

`trade_skill / 43` is the displayed 0-5 skill level, so each level is worth 2%, up to
a 10% discount at level 5 (skill byte 215). The administrator routine applies it right
after `get_buy_price` (`0x004FF944`: `price * percent / 100`, with the operand order
flipped above `0x1000000` to avoid overflowing); its sell orders are settled through
`get_sell_price` without any skill adjustment, so the discount is buying-only. An
office whose administrator index (`office+0x2F2`) is invalid pays 100%.

Office administrators do gain skill like captains do (verified in-game: a long-running
save showed administrator trade levels 1-5), even though the game never displays it -
a level 5 administrator quietly buys everything 10% cheaper.

## Running a Route Stop
The executor is `0x004D5200` (thiscall on `0x006DD728`, arguments `(ship, office)`). It
runs from the **ships tick** (`0x00506720`), not from an operation: the tick tests bit 0 of
`ship+0x136`, and `0x00518860` resolves the office for the ship's merchant and current town
through the office lookup `0x005308A0` before calling the executor on the ship's current
[route stop](./file-formats/rou.md#applied-routes-at-runtime).

The stop's instructions are executed in **two passes over the whole ware order array**,
never interleaved:

|Pass|Handles|
|-|-|
|1|selling to the town (positive price) and unloading into the office (zero price, negative amount)|
|2|buying from the town (negative price) and loading from the office (zero price, positive amount)|

Between them, `0x004D5600` calls the ship's recompute (`0x005182B0`, see
[Ships](./ships.md#crew-cutlasses-and-the-equipment-weight)) and keeps the returned **free
capacity** as a budget, clamped at zero. Pass 2 caps every purchase and every office load
against that budget and decrements it as it spends.

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
|load from the office|`amount`, capped by [`0x00500EC0`](./basics/office.md#what-the-lock-bit-does) and by the capacity budget|
|sell to the town|`0x0052EA80(ware, town, price) - town stock`, capped by `amount`|
|buy from the town|capped by `amount`, by the capacity budget, and by the merchant's cash|

A purchase the merchant cannot fully afford is **scaled down proportionally**
(`0x004D5705`) rather than skipped, and its cost runs through the captain's
[buying discount](#buying-discount). Goods entering the office go through
[`0x004FF6F0`](./basics/office.md#average-purchase-price), which is how the office's
average purchase price follows the cargo.

Finally, the stop's [action byte](./file-formats/rou.md#action-byte) is consulted twice:
bit `0x04` (the first stop of the route) builds a record from the captain's name ids and
the ship's registry id and passes it to `0x004D6530`, and bit `0x02` clears the low bits of
`ship+0x136` once the stop is finished.

## The Administrator's Trading
An office administrator is not driven by the ships tick but by the world tick itself:

```
advance_time 0x00530E80
  └─ 0x0051BA10   walk the town's offices, chaining office+0x2CA
       └─ 0x004FFF20   the per-office periodic routine, dispatching on office+0x2D6:
            bit 0x10 -> 0x004FFA30   AI-merchant offices (skipped when merchant+0x8 is 0)
            bit 0x02 -> 0x004FFC20
            bit 0x01 -> 0x004FF780   the administrator's trading
```

The town whose offices are visited comes from the tick counter (`tick >> 3`), so offices
are worked through in a staggered round rather than all at once.

`0x004FF780` walks the wares from 23 down to 0 and acts on each
[order](./basics/office.md) whose price is non-zero. On the sell side (positive price, the
minimum price) it offers the stock **above** the minimum store quantity - so that column is
a floor the administrator sells down to, not a target it tops up to. Purchases apply the
administrator's own [buying discount](#buying-discount).
