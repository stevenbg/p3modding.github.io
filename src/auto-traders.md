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
current date serial. See [Wages](#wages) for `field_C_daily_wage`. A town's tavern offers a captain
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

## Gaining and Losing Skill
Every one of the three skill bytes is written by a single operation,
[`0x12`](./operations/0012-auto-trader-skill-gain.md) (handler `0x00538A80`), and every
write goes through the same clamp: a gain is added to the current byte and the result is
cut to that skill's ceiling if it passes it or wraps.

**The ceiling belongs to the record's slot, not to the man.** It is read from the table at
`0x00673B34` - 250, 200, 250, 150, i.e. displayed levels 5, 4, 5 and 3 - indexed by
`index & 3` for navigation, `(index >> 2) & 3` for trade and `(index >> 4) & 3` for
combat, where `index` is the record's position in the auto-trader array.

A slot is settled when the record is **created**, and nothing moves a record afterwards, so
what matters is whether an action allocates a new record or reuses the existing one:

- **Dismissing a captain does not destroy his record.** The finalizer (task `0x29`,
  `0x004DDE70`) unlinks it, writes `0xFF` into its merchant byte and links it back into the
  town's chain, so the same man - same slot, same ceilings, same skills - waits in that
  tavern to be hired again. Hiring, dismissing, moving him between ships: none of it
  reallocates anything.
- **An office administrator is the exception.** Operation `0x5E` frees his record to the
  freelist when he is dismissed (`0x005098B0`, which also clears the office's autotrade
  flags), and its hire path unconditionally allocates a fresh one and writes trade `= 0`
  (`0x0053DA1D`) before recomputing the wage - so the record that comes back is a different
  man who has to earn his trade skill again from nothing. Ceilings never enter into it: his
  growth path has no ceiling table, and his navigation and combat are dead stats.

For a human player's captains the navigation ceiling does double duty as the threshold the
other two skills are tested against, which is what decides whether trade and combat ever
reach ceilings of their own; see
[the ten-day update](./scheduled-tasks/0003-ten-day-update.md#what-that-means-for-a-captains-final-skills).

**Skill is lost as well as gained.** The clamp runs for all three skills on every
application, even one whose gain for that skill is zero, so a skill sitting *above* its
ceiling is cut back by the first gain event that reaches the record. Fresh records are
rolled roughly uniformly over `0..255` per skill (sum capped at 600), so starting above a
ceiling of 150 or 200 is common: on a live save a captain carrying 253 in all three came
back as `250 / 150 / 250`, two and a half displayed levels of trade gone.

### Where gains come from
|Producer|Gain|
|-|-|
|the [ten-day world update](./scheduled-tasks/0003-ten-day-update.md)|`rand % 51` for a human player's captain, a flat `8` for an AI merchant's, about four times a year per captain|
|a pirate raider reaching its hideout (`0x00514C93`)|a flat `50`, once per crewed ship in the arriving convoy, credited to the convoy's acting ship - see [Pirates](./pirates.md#bands-and-hideouts)|

Nothing else writes a skill byte, and nothing anywhere decrements one. Nothing depends on
what the captain has been doing either, with the single exception of the hideout award:
gain is a roll, not a reward for sailing, trading or fighting. A record sitting in a
tavern gains nothing at all, because the sweep only walks merchants' ship chains - eleven
tavern pirates were byte-identical across two dumps 259 days apart.

That also makes the hideout award by far the fastest growth in the game, and the only one
a player can drive. A ship handed to a pirate captain leaves the merchant's ship chain
(its `field_0_merchant_index` becomes `0xFF`), so the ten-day sweep never sees it again
and the award is its only source of skill: one measured raider went from `60 / 164 / 186`
to all three ceilings inside 162 days.

### Administrators
An office administrator gains too, from the same ten-day sweep but on his own path -
[operation `0x67`](./operations/0067-administrator-skill-gain.md), which adds exactly 43
points, one displayed level, and stops when that would wrap. A fresh administrator starts
at `0`, so his trade skill walks `0, 43, 86, 129, 172, 215` and no further; it is always
an exact multiple of 43. His navigation and combat are never touched, and only a **human**
merchant's administrators gain at all.

An administrator is therefore much simpler than a captain: only trade does anything for him
(it is the [buying discount](#buying-discount), and he neither sails nor fights), there is
no ceiling table on his path, and there is nothing to lose to a clamp. Every administrator
who is left in place long enough ends at level 5 - the `42%` roll only decides how long that
takes, roughly two eligible rounds per level.

### Retirement
`field_4` is a birth stamp in ticks - the initializer writes `game_time - offset` with
`offset = 46720 * (48..79) + 1792 * (0..31)`, so a new record is 24.0 to 40.1 years old.
The ten-day sweep retires a captain once his age passes `0x474A00` ticks, almost exactly
**50 years**, marking the record with `field_E` and scheduling task `0x27` to take him off
his ship. An AI merchant's captain goes at once; a human player's gets a probability roll
that cannot fire before about 51.6 years, and when it does it arrives as
[operation `0x13`](./operations/0013-captain-retirement.md) and a message.

## Wages
`field_C_daily_wage` is recomputed from the skills whenever they change, by one of two
routines - which one depends on the path that touched the record, not on the kind of record
it is:

|Routine|Formula|Used by|
|-|-|-|
|`0x004FE190`|`(nav + trade + combat) / 50 + (field_8 % 11) + 10`|the [skill gain](./operations/0012-auto-trader-skill-gain.md) handler, so every captain and pirate|
|`0x004FE160`|`20 * (trade / 43) + 10`, i.e. only ever 10, 30, 50, 70, 90 or 110|the administrator paths, operations `0x5E` and `0x67`|

Both check out against live saves: a captain with `39/61/125` and `field_8` = 6 reads wage
22, a pirate with `237/251/112` and `field_8` = `0xFB` reads 31, and administrators at trade
0, 43, 86, 129 and 215 read 10, 30, 50, 70 and 110.

**What the interface shows is not the record's wage.** For an administrator the trading
office window calls `0x00500F10(office)`, which returns the record's wage **plus**
`office+0x2D2`, and falls back to `office+0x2D2 + 10` when the post is vacant - the cost of
the level 0 administrator you would get. [`office+0x2D2`](./basics/office.md) counts the
**business buildings the merchant owns in that town**, one per building - verified across
several saves and offices. So a busy office pays its administrator a gold a day more for
every business it runs, and an office with none pays exactly what the record says.

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

Office administrators gain trade skill like captains do, in whole displayed levels
(see [Administrators](#administrators) above), even though the game never shows it - a
level 5 administrator quietly buys everything 10% cheaper.

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
