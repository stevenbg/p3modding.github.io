# Skill
Every [auto trader](../auto-traders.md) carries three skill bytes - `field_9` navigation,
`field_A` trade, `field_B` combat - which the interface shows as 0 to 5, **one step per 50
raw points**. This page is the whole story of how they move.

Beware a second, coarser step of **43** that looks like the same thing and is not. The
tavern's captain-offer panel divides each of the three bytes by 50 (`0x005CFA50`,
`0x005CFB11`, `0x005CFBD5` - the magic `0x51EB851F` with `sar edx,4`, and note it prints
**trade first**, then navigation, then combat), while the buying discount and the
administrator wage divide the trade byte by 43 (`0x004D5347`, `0x004FE160`: magic
`0x2FA0BE83` with `sar edx,3`). So a captain the panel shows as trade 2 is already on the
third pricing step, and the two numbers part company from 43 raw points upward.

## One Writer, One Clamp
All three bytes are written by a single operation,
[`0x12`](../operations/0012-auto-trader-skill-gain.md) (handler `0x00538A80`), and every
write goes through the same clamp:

```python
new = (skill + gain) & 0xFF          # a byte addition
if new > cap or new < skill:         # over the ceiling, or wrapped
    skill = cap
else:
    skill = new
```

Nothing else in the game writes a skill byte, and **nothing anywhere decrements one** -
every loss of skill is this clamp firing. The handler finishes by recomputing the
[wage](../auto-traders.md#wages) from the new skills (`0x004FE190`).

## The Ceiling Belongs to the Slot
The three ceilings come from the four-byte table at `0x00673B34` - **250, 200, 250,
150**, i.e. displayed levels 5, 4, 5 and 3 - indexed by bits of the record's **index in the
auto-trader array**:

|Skill|Table index|
|-|-|
|navigation|`index & 3`|
|trade|`(index >> 2) & 3`|
|combat|`(index >> 4) & 3`|

So a captain's ceiling in each discipline belongs to the array slot, not to the man. The
slot is settled when the record is created and nothing ever moves a record, so a captain
keeps his ceilings for life - and because records are recycled through the freelist, a
re-hired [administrator](./administrators.md) inherits whatever the allocator hands out.
Which actions reuse a record and which allocate a new one is in
[Array Layout and Slot Recycling](../auto-traders.md#array-layout-and-slot-recycling).

Two slot patterns are worth knowing: `index & 0x15 == 0` gives 5 stars in all three
disciplines, and `index & 0x3F == 0x3F` gives 3 stars in all three.

Throughout the rest of this page **`T`** means the record's *navigation* ceiling, because
that one number does double duty: it is also the threshold the ten-day sweep tests the
other two skills against.

## Trade and Combat Share One Gain Field
`0x00538B16` and `0x00538B2E` are the same three instruction bytes: the handler reads
payload `+0x0C` for the trade gain and reads it again for the combat gain. The payload's
fourth dword at `+0x10` is never read here and never written by either producer.

So **trade and combat always move by the same amount.** The difference between them is
fixed when the record is created, and nothing but a clamp can change it: whichever of the
two is higher reaches its ceiling first and waits there while the other catches up. The
same applies in reverse - a gain aimed at trade raises combat even when combat is already
past the threshold that would have refused a combat roll, which is what lets the pair climb
well beyond that threshold.

Measured on a live save: `trade - combat` was unchanged for every captain across two dumps
259 days apart, except where one of the two had met its ceiling in between.

### The Threshold and the Ceilings Are Separate Tables
The ceiling table exists twice, byte-identical (`250, 200, 250, 150`), and each copy
has exactly one consumer - a full-executable cross-reference finds no other reads:

|Table|Read by|Sites|
|-|-|-|
|`0x00672824`|the sweep's **gate** (human branch only), always indexed with the navigation bits|`0x004DD0EC`, `0x004DD0F7`|
|`0x00673B34`|the gain handler's **clamp**, indexed per skill|`0x00538AE7`, `0x00538B10`, `0x00538B37`|

So the threshold `T` and the real ceilings are independently patchable: raising the
gate table alone stops the sweep refusing trade and combat rolls while the clamp
still enforces every skill's own ceiling - which is what `mod-fix-captain-skill-cap-gate`
does. Since a refused roll and a clamped-to-ceiling gain have the same outcome, that
is behaviourally a correct per-skill gate.

## Where a Captain Ends Up
For a human player's captains the [ten-day sweep](../scheduled-tasks/0003-ten-day-update.md)
only pays out while the skill it happens to test is below `T`. Combined with the shared gain
field, that decides where a career stops.

Navigation is simple: its rolls fire while navigation is below `T`, and `T` is also
navigation's own ceiling, so navigation ends at exactly `T`.

Trade and combat are not. A roll on either pays **both**, so the pair keeps growing while
the **lower** of the two is below `T` - the laggard's rolls carry the leader along, past `T`
and on toward the leader's own ceiling, where the clamp stops it. Once both are at or above
`T` neither branch can fire again and both freeze wherever they stand. The last gain before
that comes from just under `T` and is at most 50, so the lower of the two ends somewhere in
`T`..`T+49` and stays there for the rest of the captain's life.

One slot therefore produces very different careers. Take a record whose navigation ceiling
is 150 and whose trade and combat ceilings are both 250:

|Skills as created|Where they end up|
|-|-|
|trade and combat both low|they cross 150 together and stop between 150 and 199 - a displayed 3, never the 5 their own ceilings would allow|
|trade 170, combat 15|combat's rolls keep paying trade, which reaches its 250 ceiling and is clamped there, while combat is dragged up to 150..199 - so trade does finish at level 5|

Measured on a live save: a captain with `T` = 150 took combat from 205 to 240 over nine
months purely because his trade was sitting at 16, and both will stop the moment that trade
reaches 150.

An **AI merchant's** captain is not gated at all - his three skills simply rise until each
meets its own ceiling.

## A Skill Above Its Ceiling Is Pulled Down
The clamp is unconditional and runs for all three skills on every application - including
one whose gain for that skill is `0`. An over-ceiling skill is therefore cut to its ceiling
by the first operation `0x12` that reaches the record, whichever skill the gain was
actually meant for.

That is not a corner case. The record initializer `0x004FDF50` produces each skill by
reinterpreting the bits of a float (`0x004FE046`, `0x004FE073`, `0x004FE0AE`), giving a
roughly uniform `0..255` per skill with the sum capped at 600, so a fresh record above 150
or 200 is common. Measured on a live save: a captain with all three skills at 253 was cut
to `250 / 150 / 250` the first time the sweep reached him, losing 103 raw points - two
displayed steps - of trade. A captain can visibly drop from a displayed 5 to a 3 shortly
after being hired.

## Where Gains Come From
Two producers, and only two:

|Producer|Gain|
|-|-|
|the [ten-day world update](../scheduled-tasks/0003-ten-day-update.md)|`rand % 51` for a human player's captain, a flat `8` for an AI merchant's, about four times a year per captain|
|a pirate raider reaching its hideout (`0x00514C93`)|a flat `50`, once per crewed ship in the arriving convoy, credited to the convoy's acting ship - see [Bands and Hideouts](../pirates/bands.md)|

Nothing depends on what the captain has actually been doing, with the single exception of
the hideout award: gain is a roll, not a reward for sailing, trading or fighting. A record
sitting in a tavern gains nothing at all, because the sweep only walks merchants' ship
chains - eleven tavern pirates were byte-identical across two dumps 259 days apart.

That also makes the hideout award by far the fastest growth in the game, and the only one a
player can drive. A ship handed to a pirate captain leaves the merchant's ship chain (its
`field_0_merchant_index` becomes `0xFF`), so the ten-day sweep never sees it again and the
award is its only source of skill: one measured raider went from `60 / 164 / 186` to all
three ceilings inside 162 days.

## The Buying Discount
Auto traders buy cheaper as their trade skill grows, which is what the trade skill is
*for*. The captain (`0x004D5347`) and administrator (`0x004FF7E8`) buying routines both
compute the percentage of the transaction price to pay from `field_A_trade_skill`:

```
percent_paid = 2 * (50 - trade_skill / 43)
```

The step here is **43, not the display's 50**, so each 43 raw points is worth 2% and the
full 10% discount arrives at skill byte 215 - while the interface still shows that captain
as a 4. The administrator routine applies it right after
`get_buy_price` (`0x004FF944`: `price * percent / 100`, with the operand order flipped
above `0x1000000` to avoid overflowing); its sell orders are settled through
`get_sell_price` without any skill adjustment, so the discount is **buying-only**. An
office whose administrator index (`office+0x2F2`) is invalid pays 100%.

Office administrators gain trade skill in whole 43-point steps even though the game never
shows it, so an administrator at 215 quietly buys everything 10% cheaper - see
[Administrators](./administrators.md).
