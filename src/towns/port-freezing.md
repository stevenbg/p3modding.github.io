# Port Freezing
Every winter the northern and eastern ports ice over. A frozen port turns arriving
ships away, and the town is treated as a town in crisis while it lasts.

The flag is **bit `0x04000000` of the town flags at `town + 0x2C8`**. Three separate
sites agree on that meaning:

- the [unfreeze task](../scheduled-tasks/0035-unfreeze-port.md) clears it and posts
  *"The port of %s is open again."*;
- the ice pass below sets it and posts *"The port of %s is frozen."*;
- `update_town_price_thresholds` reads it at `0x005280B7` as part of the crisis mask
  `0x04000A10`, alongside siege, blockade and pirate attack (see
  [Thresholds](./ware-prices/thresholds.md)).

That last one is why a search for the bit on its own comes up short: the constant in
the code is the combined mask, never `0x04000000` alone.

## The Ice Fields
Four more town fields drive the model:

|Offset|Type|Meaning|
|-|-|-|
|`0x9B8`|u32|accumulated cold. The whole model is a function of this|
|`0x9BC`|u8|**yesterday's** ice level, copied from `+0x9BD` at the top of each pass - or forced to `5` when `+0x9BD` had bit `0x80` set|
|`0x9BD`|u8|**today's ice level** in the low 7 bits; bit `0x80` marks the town as able to freeze today|
|`0x9C0`|u32|ice effect counter, reduced by `0x100` per pass|
|`0x9C4`|u8|ice effect flags; bit `0x4` = the ice effect is running: set when the level rises above yesterday's (`0x004E486B`) and held while `+0x9C0` is positive (`0x004E47AB`), cleared when that counter drains (`0x004E47BB`) and by the day-58 reset|

## The Daily Pass
Scheduled task `0x0D` (`0x004E4984`) is the weather tick. It looks at the day of the
year and calls the ice pass `0x004E45C4` when that day is `<= 58` or `>= 333`,
otherwise the unrelated routine `0x004F34A4`.

**The pass runs once a day.** The handler reschedules itself: its tail adds `0x100`
ticks - one day - to its own due stamp (`0x004E4A2A`), and `0x004E4984` has a single
call site in the task dispatcher. Per pass and per day are therefore the same thing
everywhere below. Measured over thirteen days of the decay arm, every town's daily
cold loss came out at `1.002` times its own rate, across rates spanning a factor of
six - exactly the decay law's mean.

Its random source is a stack LCG (`0x0046C380` returns `state % bound`) seeded from
four live fields of the town array, so the weather is **deterministic given the world
state** rather than free-running.

### The per-climate rate
Each town has a climate id at `[0x006DE4B8 + town_index]`, indexing 52-byte blocks at
`0x006DDBB0`. The pass reduces the first two dwords of the block to a single number:

    rate = (block[0] - 2 * block[1] + 3600) / 50

The divisor is 50, not the more common 100: the reciprocal at `0x004E4696` is
`0x51EB851F` with a total shift of 36 (a shift of 37 would be `/100`). Everything else
scales with the rate. On the standard map it runs from `16` (Bruges) to `106` (Ladoga),
rising west to east - doubled from probe values that were computed with a `/100`
divisor, so integer truncation can put a true value one higher than shown.

### Three arms, not one season
The day of the year picks one of three behaviours at `0x004E46AE`. This is finer than
the season gate above, and it is what confines freezing to the first month:

|Day of year|Arm|Effect on `+0x9B8`|
|-|-|-|
|`>= 333` or `< 32`|accumulate (`0x004E4762`)|`cold += (rand(60) + rate) / 3`|
|`32`..`57`|decay (`0x004E4708`)|`cold -= (rand(rate) + 2.5 * rate) / 3`, mean `rate`; clamped at 0|
|`58`|reset (`0x004E46BE`)|`cold = 0`, `+0x9C0 = 0`, `+0x9C4 &= ~4`|
|`59`..`332`|-|the pass is not called at all|

The `2.5` is the double at `0x00672848`, subtracted from the random draw. The decay arm
was measured against the formula and matches to 0.2%.

So cold climbs from early December, **peaks around day 31**, and is drained away
through February. A port can only freeze while its level is still high.

### The ice level
After the accumulate or decay step, in byte registers (`0x004E47DF`):

    level = ((cold >> 9) & 0xFF) + 2
    if (level > 5) level |= 0x80        // the town may freeze this pass

and the result is stored in `+0x9BD`. One exception: with `cold == 0` on a pass that
runs *outside* its own winter window, the pass writes level `1` and stops. That window
mismatch happens on day 58 alone - the caller admits `day <= 58` while the pass itself
counts `day >= 58` as not winter - and the `1` then sits there untouched until day 333.

Because the level and the "may freeze" bit share one byte, a level above `0x7E` aliases
into the bit. No town on the standard map gets near that.

The level byte was checked against this arithmetic on every town of two different saves
across nine samples, including the aliasing values, and matched every time.

### Freezing
A town that may freeze and is not already frozen then takes a **`rand(0x6400) < 0x180`
roll - 1.5% - per pass** (`0x004E4872`). On a hit the pass:

1. sets `0x04000000` in `town + 0x2C8`;
2. allocates a scheduled task (`0x004D8CF0`) with opcode `0x35` at `+0x6` and the town
   index at `+0x8`, due `[0x006DE4B4] + ((level & 0x7F) + 1) << 8` - so the thaw is
   **level + 1 days** away, and a colder port stays shut longer;
3. posts *"The port of %s is frozen."*

## Which Ports Freeze
Being able to freeze at all needs `cold >= 0x800` (2048), the point where the level
passes 5. With the measured accumulation that threshold lands at a **rate of about 72**,
and the game agrees exactly: sampled at the day-30 peak, the towns marked as able to
freeze are precisely the eight whose rate is `74` or more, with a clean gap to the
ninth.

|Town|rate|
|-|-|
|Ladoga|106|
|Novgorod|100|
|Reval|90|
|Stockholm|84|
|Bergen|80|
|Oslo|76|
|Riga|76|
|Visby|74|
|*Aalborg, the next town down*|*56*|

Sampled earlier in the accumulation, at day 21, only the four towns with a rate of `84`
or more had crossed - the same test at a different point, with the same result.

**There is no per-town "can freeze" flag and no region list.** Which harbours ice over
falls out of the climate rate against one fixed threshold, which is why the answer is a
band of northern and eastern ports rather than an authored set.

## What a Frozen Port Does
- **Ships are turned away.** The ships tick's destination check treats a frozen port
  exactly like a blockaded one; see
  [A Closed Destination Port](../ships/trade-routes.md#a-closed-destination-port). A ship
  whose only remaining destination is the frozen town ends up with nothing to do and
  reports *"%s's trade route: no destination specified"*.
- **Prices behave as in any crisis**: `t1` stretches to 28 days, the building-material
  factor doubles and the food demand bitmask is added
  ([Thresholds](./ware-prices/thresholds.md)).
