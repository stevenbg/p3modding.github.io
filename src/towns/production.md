# Production
A town makes goods in its own right, separately from anything a merchant builds there. The
market hall's Production page shows both side by side, and they come from two different
objects through the same field.

## The Town and Trader Columns
Both towns and [trading offices](../merchants/trading-office.md) begin with a
[storage](../reference/storage.md) struct, so both have a `field_C4_daily_production` array
at `+0xC4`. That one offset is the whole distinction:

|Column|Source|
|-|-|
|Town|`town + 0xC4 + ware*4`|
|Trader|the town's office chain - head `town + 0x784`, next `office + 0x2CA`, bounded by the office count `[0x006DE4A8]` - summing `office + 0xC4 + ware*4`|

`ui_prepare_market_hall_window_production_page` (`0x005DE960`) reads the town value at
`0x005DEA3D` (barrel wares) and `0x005DEA98` (bundle wares), and accumulates the office
values at `0x005DE9CD`, multiplying each by 7 for the week at `0x005DE9DB`. The town value
never gets that `*7`, which is the
[market hall production bug](../bugs/market-hall-production-town.md).

So the Town column is genuinely the town's own output: no merchant building of any owner
contributes to it, because merchant buildings deposit into their own office.

## Two Production Arrays, One Term Apart
Both are **daily accumulators**. The town tick zeroes `+0x64`, `+0xC4` and `+0x310` per ware
and `memset`s `+0x490` (`0x0051BAFF`), then runs the town's facilities; each town ticks once
a day, so both arrays hold a day's worth.

|Field|Holds|
|-|-|
|`town + 0x0C4`|**actual** output of the town's own facilities, scaled by staffing|
|`town + 0x490`|**nominal** capacity at full staffing, of *every* facility in the town including merchant-owned ones|

They are written 30 bytes apart in the same routine and differ in exactly one term:
`0x0050EB4A` accumulates `efficiency * full_workforce` into `+0x490`, while `0x0050EB5D`
accumulates `employees * efficiency` into `+0xC4` (as a read-modify-write pair with the stock
at `+0x4`, `0x0050EBC1`). `full_workforce` is the facility type's entry in the
[u8 table at `0x006735E8`](../reference/facilities.md#full-workforce-per-type) - a constant -
which is the whole reason `+0x490` ignores staffing.

That `+0x490` covers merchant facilities too is measurable. Brickworks is identical in every
town that has one - 27 employees - so `+0x490 - +0xC4` isolates the merchants' share:

|town|`+0xC4`|`+0x490`|difference|traders' actual output|
|-|-|-|-|-|
|Rostock|2713|2713|0|0|
|Scarborough|2713|5728|3015|3015|
|Groningen|3618|7638|4020|4020|
|Bruges|2713|5728|3015|1507|
|Oslo|3618|7638|4020|**0**|

The difference equals the traders' actual output exactly where their buildings are fully
staffed, and exceeds it where they are not - Oslo has 4020 of merchant brick capacity
producing nothing at all. `+0x490` is what the
[ware price thresholds](./ware-prices/thresholds.md) anchor to, which is why an unstaffed
building still deepens them.

A consequence worth stating: `+0xC4 / +0x490` is **not** a utilization figure. It is the
town's actual output over everybody's nominal capacity.

## What a Facility Produces
A town's facilities live in a **21-slot array at `town + 0x840`**, stride `0x10`, walked by
the town tick at `0x0051BB72` and by world setup at `0x00545EE4`. The array is **indexed by
facility type**: the constructor `0x005100F0` writes the slot index into `field_6_type`, so
slot `n` is always type `n`. See [Facilities](../reference/facilities.md) for the types.

Output per ware per day is

```
amount = employees * efficiency / k
```

with `k` fixed per facility type. Measured across all 21 slots of 24 towns in a live save:

|type|ware|k|type|ware|k|
|-|-|-|-|-|-|
|Sawmill|Timber|7.68|Saltworks|Salt|30.72|
|Brickworks|Bricks|7.64|IronSmelter|PigIron|30.72|
|FishermansHut|Fish|15.36|FarmSheep|Wool|30.72|
|FarmGrain|Grain|15.36|Workshop|IronGoods|51.20|
|Brewery|Beer|21.79|WeavingMill|Cloth|51.20|
|FarmCattle|Meat|61.44|Apiary|Honey|76.8|
|HuntingLodge|Skins|153.6|Pottery|Pottery|76.8|
|FarmCattle|Leather|153.6|Pitchmaker|Pitch|76.8|
||||Vineyard|Wine|76.42|

Whale oil is not in the table because it does not follow the formula - see
[Whale Oil Has No Facility](#whale-oil-has-no-facility).

The clearest demonstration is Brickworks, which has **27 employees in every town that has
one**: its output takes exactly two values, 2713 and 3618, tracking its efficiency of 768 or
1024 in that exact ratio.

### The Crops Scale k By Three Town Flag Bits
`k` is not literally hardcoded. Each type's
[producer routine](../reference/facilities.md#producing-a-ware) folds a small integer factor
into its divisor, and **four of the twenty-one read that factor from `town + 0x2C8`**:

|Producer|Ware|factor|divisor|`k` at the default factor|
|-|-|-|-|-|
|`0x0050EAD0` FarmGrain|Grain|`6`, or `4` with bit `0x2`; then `/3` with bit `0x2000`, or `x2` with bit `0x4000`|92.16|15.36|
|`0x0050EA00` Apiary|Honey|`4`, or `2` with bit `0x2`; then `/2` with bit `0x2000`, or `x3` with bit `0x4000`|307.2|76.8|
|`0x0050F100` Vineyard|Wine|`4`, or `2` with bit `0x2`; then `/2` with bit `0x2000`, or **`x2`** with bit `0x4000`|-|76.42|
|`0x0050EBF0` FarmHemp|Hemp|`3` with bit `0x2`, else `4` with `0x2000`, else `9` with `0x4000`, else `6`|368.64|61.44|

The Vineyard shares the Apiary's base and its `0x2000` branch but **not** its `0x4000` one:
where the Apiary triples with `lea ebx,[ebx+ebx*2]` (`0x0050EA49`), the Vineyard doubles
through the 64-bit helper `0x0063A6A0` (`0x0050F162`). It runs its whole calculation through
those helpers, with further factors of `0x43` and `5`, which is why its `k` is 76.42 rather
than the Apiary's 76.8.

The other seventeen producers never read the field - checked over the full extent of all
twenty-one routines, so grain, honey, wine and hemp are the complete set. That is exactly the
four **crops**: the animal farms, the fisherman's hut and every industry are outside it.

Every `k` in the table above is therefore the value with all three bits clear, which is what
all 24 towns of the measured save had. **What the three bits mean is not yet established** -
a harvest or seasonal modifier on crops is the obvious candidate and is untested.

### Facilities That Make Two Wares
Two types produce a second ware, and the producer gates that second add on a `flag` argument
the dispatcher computes.

- **FarmCattle** always produces meat *and* leather, in all 14 towns that have one, at a
  ratio of exactly **5:2**. Its flag is a live `leather_stock < t3` test (`0x0051049C`).
- **FishermansHut** produces fish everywhere, and in 5 of its 11 towns whale oil as well. Its
  flag is a constant `1`, and the producer then skips the whale oil add whenever
  `whale_oil_stock >= t2` (`0x0050E852`).

In those 5 towns the hut's own efficiency is 706 rather than 942 - the low grade - and that is
**caused** by the whale oil, not correlated with it: see below. The per-worker fish rate is
identical in all eleven towns, so what a whale-oil town loses is the efficiency tier, not the
rate.

## Whale Oil Has No Facility
Whale oil is the one ware with no entry in the facility array. Its slot in the
[ware-to-type table](../reference/facilities.md#which-ware-a-facility-produces) at
`0x00672C88` is the sentinel `0xFF`, and its productivity sits in a standalone town field,
`town + 0x2CC`, on the same `1024 / 768 / 0` scale as a facility's `field_8_productivity`.

Town setup writes it at `0x00545961`, from the bit **immediately after** the 17 that cover the
facility types - `0x20000` in the same two bitmaps:

|whaling bit is in|`town + 0x2CC`|FishermansHut `field_8`|
|-|-|-|
|the effective bitmap|`1024`|forced to `768`|
|the ineffective bitmap|`768`|forced to `768`|
|neither|`0`|left as the bitmaps put it|

**A whaling town can therefore never have effective fish**: the same branch that enables
whaling writes `town + 0x898` down to `768`. That is the mechanism behind the 5 towns above,
and it predicts their efficiency exactly - `942 * 768 / 1024 = 706`.

`town + 0x2CC` then stands in for the missing facility's *efficiency* as well. The producer
`0x0050E690` computes the two wares from different sources off the same workforce:

|Ware|Nominal, `town + 0x490`|Actual, `storage + 0xC4`|
|-|-|-|
|Fish|`efficiency * n / 15.36`|`employees * efficiency / 15.36`|
|WhaleOil|`town[0x2CC] * n * 27 / 1024`|`employees * town[0x2CC] * 27 / 1024`|

with `n` = 65, or **72** in a whaling town (`0x0051032C`, and the matching `0x48` override on
the worker cap at `0x0051016A` - the only entry of the
[full-workforce table](../reference/facilities.md#full-workforce-per-type) the game adjusts at
runtime). The hut also consumes `fish/5` salt and `fish/10` hemp.

So whale oil has **no `k` and no efficiency**: its rate is a flat `27/1024` per unit of
productivity, i.e. **27 per worker** at the effective grade and 20.25 at the low one. All five
measured towns read 27 per worker, so all five carry `town + 0x2CC = 1024` - whaling effective
while their fish is forced low. A town with whaling in the *ineffective* bitmap has not been
observed.

Note what whale oil does and does not borrow from the fisherman's hut. It takes the hut's
**employees** - so an unstaffed hut makes no whale oil either - but never its
`field_0_efficiency`, which `town + 0x2CC` replaces. Changing the hut's efficiency moves its
fish and leaves its whale oil untouched.

The same split shows in the interface. The town information window's two produced-ware lists
read `field_8_productivity`, never `field_0_efficiency`, comparing against `900` for effective
and `200` for low - which is also why `768` and `683` collapse into one grade. When the
ware-to-type lookup yields `0xFF` they fall through to `town + 0x2CC` against a threshold of
`1000` instead (`0x005B7E41` and `0x005B7FA1`), so whale oil is listed on the strength of a
field no facility owns.

## Effective and Ineffective Production
A town produces only some wares, and each of those is either **effective** or
**ineffective**. That state is `field_8_productivity` on the facility, and it holds exactly
three values:

|`field_8_productivity`|Meaning|
|-|-|
|`1024`|effective|
|`768`|low, at three quarters|
|`683`|low, at two thirds|
|`0`|the town has no such facility|

`field_0_efficiency` is the number the output formula uses, and it is derived from
`field_8_productivity` - but **the multiplier between them is not a constant of the
executable.** World setup computes, at `0x00545E48`:

```
field_0_efficiency = BASE_EFFICIENCY[type] * field_8_productivity / 1024
```

where `BASE_EFFICIENCY` is a u16 table of 21 entries at `0x00673C24` indexed by facility type,
terminated by a `0` - the only reader of that address in the executable. That relation was
verified exactly on all 255 populated facility slots of one 24-town save.

**The table is a default, and a scenario may author its own values instead.** Measured:

|world|slots matching `BASE_EFFICIENCY[type] * prod / 1024`|productivity values present|
|-|-|-|
|a fresh open-ended game, at each of the five difficulty presets|**255 / 255**|`1024` x195, `768` x60|
|"Rise of the Hanseatic League", 1362|**255 / 255**|`1024` x195, `768` x60|
|"The Flying Trader", a **fresh** 1305 start|**106 / 211**|`1024` x162, `768` x16, `683` x33|

The first two are identical, so that campaign is played on the standard generated world. The
Flying Trader is a genuinely custom scenario - 22 towns, and roughly half its facilities set
to something other than the table - and it is authored that way from the first day, not drifted
into during play.

A town created **at runtime** always gets the default: a player-founded settlement goes through
the create-a-town routine `0x00531D50`, its per-town setup `0x00545CB0` and so the seeding at
`0x00545E48`, which reads the table. In a played Flying Trader save the one town matching the
table was exactly the one the player had founded, its Militia efficiency `0` rather than `1024`
because a new settlement has not raised one.

**Difficulty has no effect.** Five fresh games, one per preset, produced identical efficiencies
and an identical productivity distribution.

So: the table tells you what world generation and a newly founded town use. For a town that
came with a custom scenario, **read `field_0_efficiency` rather than computing it.**

|type|base|type|base|type|base|
|-|-|-|-|-|-|
|Militia|1024|Apiary|1229|IronSmelter|1024|
|Shipyard|1024|FarmGrain|1331|FarmSheep|1229|
|[Construction](./construction.md)|1024|FarmCattle|1638|Vineyard|4096|
|Weaponsmith|1024|Sawmill|1229|Pottery|1024|
|HuntingLodge|2048|WeavingMill|1536|Brickworks|1024|
|FishermansHut|942|Saltworks|1024|Pitchmaker|1024|
|Brewery|1638|Workshop|2048|FarmHemp|1024|

`field_8_productivity` is a per-town, per-ware value in the savegame, and it is **entirely
static**: identical across the five difficulty presets, identical in July and January of the
same game, identical between a fresh scenario start and a played save of it. Nothing in the
production code reads the calendar - the only month read anywhere nearby is in
`update_town_price_thresholds` (`0x00528070`), so the season affects prices, not output.

The town information window shows only two grades, **effective** and **low**, so both `768`
and `683` display as low. In output terms the low grades cost a quarter and a third
respectively, and a `683` facility produces 88.9% of what a `768` one would.

Which grades a world uses is scenario data. Measured across eight worlds:

|world|towns|productivity values|
|-|-|-|
|a generated game, at each of the five difficulty presets|24|`1024` x195, `768` x60|
|"Rise of the Hanseatic League", "The Advancement", "Reorganization", "Poor Harvest", "The Fire", "The Black Death"|24|`1024` x195, `768` x60|
|**"The Flying Trader"**|**22**|`1024` x162, `768` x16, **`683` x33**|

So the generated world and six of the seven campaigns share one identical production layout,
and the `683` grade appears in exactly one bespoke scenario - where it falls on the bulk goods
(grain, meat and leather, wool, hemp, skins, pig iron, bricks) while `768` falls on honey,
timber, wine and pottery, with no ware ever using both. Note that "Poor Harvest", a scenario
named for bad yields, uses the standard layout untouched: the game does not model a poor
harvest by lowering this field.

### Where Productivity Comes From
`0x005458D0` (thiscall on the town, two dword arguments) writes the whole array from **two
bitmaps**:

```
slots 0..3               -> 1024          the four municipal facilities, always effective
slots 4..20, bit n-4:                     17 iterations, 0x00545912
    effective  & bit     -> 1024
    ineffective & bit    ->  768
    neither              ->    0
bit 0x20000              -> whaling, into town+0x2CC rather than a slot
bit 0x40000              -> calls 0x005462F0(4); unidentified
```

Bit `n` therefore means facility type `n + 4`, the first ware producer being HuntingLodge at
`0x04`, and the walk is literally 17 iterations of stride `0x10` from `town + 0x888` with a
mask that doubles each pass, the four municipal slots having been written just before at
`0x005458EF`. The two bits past the end of that range are not facilities: `0x20000` is
[whaling](#whale-oil-has-no-facility) and `0x40000` is something else again. Both bitmaps
arrive in a 16-byte town record - effective at `+0x4`, ineffective at
`+0x8`, town index at `+0xE` - handed to the create-a-town routine `0x00531D50`, which also
grows the towns array, runs per-town setup `0x00545CB0` and spawns the town's
[captain and pirate](../auto-traders.md#captain-and-pirate-spawning).

For a settlement founded through an alderman mission the effective bitmap is built from the
Hanse's shortages by `determine_new_settlement` (`0x00532E30`), using the
[ware-to-facility table](../reference/facilities.md#which-ware-a-facility-produces) at
`0x00672C88` - the source of the
[new settlement ware production bug](../bugs/new-settlement-ware-production.md), whose
off-by-one is a `-3` where this bit numbering needs `-4`.

## Merchant Buildings
A merchant's production buildings - the player's included - are **not** entries in the town's
facility array. They live in one world-wide array, reached per office rather than per town:

|What|Where|
|-|-|
|array base|`[0x006DE510]`|
|live slots|the word at `[0x006DE4A6]` - 128 and 256 observed in two saves, so it grows|
|stride|`0x14`|
|resolver|`0x005303B0`: `[0x006DE510] + index * 0x14`, with **no** bounds check - every caller tests the index against the count itself|
|chain head|[`office + 0x2CC`](../merchants/trading-office.md)|
|chain next|`record + 0x8`, the chain ending at the first index that reaches the count|

Those two globals are fields of the game world object at `0x006DE4A0` - `+0x70` and `+0x6` -
which is also where the towns array (`+0x68`, `[0x006DE508]`) and the offices array (`+0x74`,
`[0x006DE514]`, stride `0x44C`, count `+0x8`) hang off. `0x004DE63B` is the canonical walk of
one office's chain.

The record has **no owner field**: which merchant owns a building is implied by whose office
chain holds it, so the player's buildings are the ones reachable from the player's offices.
One record **aggregates every building of one type that one merchant owns in one town**, and
its first 16 bytes mirror a town facility:

|Offset|Meaning|
|-|-|
|`+0x0`|u32 efficiency|
|`+0x4`|u16 workers currently employed, never above `+0xA`|
|`+0x6`|u8 facility type, only ever `0x04`..`0x14` - a merchant cannot own the four municipal types|
|`+0x7`|u8 town index|
|`+0x8`|u16 next record in the office's chain|
|`+0xA`|u16 total worker capacity|
|`+0xC`|u16 summed alongside the employees by the walk at `0x004DE668`; equal to `+0xA` in 183 of 185 measured records|

`+0xA` is the per-building worker capacity times the number of buildings. That capacity is
**30 for most types and 15 for Brickworks and Pitchmaker**, so the building count is
`+0xA / capacity` - observed totals are 15, 30, 45, 60, 90, 120, 180, 210 and 300. The 15-worker types are what make the divisor matter: six brickworks give `+0xA` = 90 and the +6% bonus, where dividing by 30 would wrongly read three buildings and +3%.

### The Same-Type Bonus
Efficiency is **not** taken from the `BASE_EFFICIENCY` table that town facilities use. A
merchant building starts from a type-independent `1024` when the ware is effective in that
town and `768` when it is not - it **inherits the town's own effective/ineffective state** for that ware, i.e. the base equals the town facility's `field_8_productivity` for the same type. Measured on 154 building records across two saves the base matched the town's facility every time, with no contradiction and no case of a merchant owning a type the town itself lacks. On top of that it gains a bonus for owning several of the same type:

|Buildings of that type|Bonus|Effective|Ineffective|
|-|-|-|-|
|1-2|+0%|1024|768|
|3-5|+3%|1054|791|
|6-8|+6%|1085|814|
|9 or more|+10%|1126|844|

The result is truncated, so `1024 * 1.03 = 1054.72` becomes `1054` and `1024 * 1.06 = 1085.44` becomes `1085`. Verified on 80 building
records: every one matched its capacity's implied count. Verified again by counting buildings
in a town where only one merchant had an office - capacities of 120, 90, 60, 60 and 60
correctly predicted 4 workshops, 3 cattle farms, 2 sheep farms, 2 fisherman's huts and 2
sawmills. And verified by construction: completing a sixth brickworks took `+0xA` to 90 and
the efficiency to 1085.

The bonus follows the buildings owned, not the workers employed - a record observed at
`+0xA` 45 with only 30 of those posts filled still carried the full +3%.

Output follows the same rule as a town facility, `employees * efficiency / k`, and the
building window shows it weekly in display units. Worked example, checked against the
interface: three fully staffed apiaries, 90 workers, honey (`k` = 76.8, a barrel ware):

```
efficiency = 1024 * 1.03            -> 1054
raw/day    = 90 * 1054 / 76.8       -> 1235.16
weekly     = 1235.16 * 7 / 200      -> 43.2
```

The window reads 43.2.

Whether a town's own facility or a merchant's building is the more productive per worker is
simply a comparison of their two `field_0_efficiency` values, and for an established map town
it is not decided by the `BASE_EFFICIENCY` table - see the caution above. Measured in one such
town: a Vineyard facility at efficiency 1126 against merchant vineyard records at 1054 and
1085, i.e. near parity. In a scenario that does not override the table the same comparison
would favour the town heavily, since its Vineyard would run at 4096.

## Weapons Are Not Like the Rest
The Weaponsmith (type `0x03`) is the only production a town has that no merchant can build,
and it runs on machinery of its own. Its dispatch branch `0x005102C2` makes two calls: it asks
`0x00510AA0` **which single armament to work on**, then hands that answer to the producer
`0x0050F6E0`. Everything the smithy actually makes on a given day is that one item.

### The Daily Figure Is Not the Output
Every day, unconditionally, all four hand weapons get

```
RATE[weapon] * employees * efficiency >> 16
```

added at `0x0050F766`, with `RATE` the four signed bytes at `0x006735E4` - **32, 40, 26, 20**
for Sword, Bow, Crossbow and Carbine. Both other terms are pinned by the game rather than by
the town:

- **efficiency**: the producer writes `1024` into `field_0_efficiency` itself whenever it
  finds it at `0` (`0x0050F724`), so the Weaponsmith's 1024 is a hardcoded default, not
  scenario data like every other facility's;
- **employees**: the branch ends by driving the count toward
  [`NOMINAL_WORKFORCE[3]`](../reference/facilities.md#full-workforce-per-type) `= 5`.

At 5 and 1024 that gives `2.5`, `3.125`, `2.03` and `1.5625`, truncated to **2, 3, 2 and 1** -
the per-day figure measured in all 24 towns, and identical in all of them for that reason.

But that loop writes **only** `storage + 0xC4 + ware*4`, the daily-production array the town
tick zeroes again next morning. It never touches the stock at `storage + 0x4 + ware*4`.
Weapons are the one case where the Production figure is a **reported rate that no goods
follow**: nothing is added to `+0x490` either, so the [price
thresholds](./ware-prices/thresholds.md) never see it. What actually accrues is one weapon a
day, below.

### The One Item That Is Really Made
The choice from `0x00510AA0` decides it, and there are two ways it is reached:

1. **Priority.** `town + 0x6D0` holds a ware id. If it falls inside the eligible range it is
   returned unchanged (`0x00510B25`) - this is the Priority radio on the Weapons smith
   window, and it wins outright.
2. **Otherwise, scarcity.** The routine scores every eligible armament as
   `stock * scale / requirement` and returns the lowest, or `0xFF` if nothing scored. Each of
   the three kinds has its own requirement, and they are not on the same scale:

|Kind|Stock read from|Requirement it is measured against|Scale|
|-|-|-|-|
|hand weapon|`storage + 0x4 + ware*4`|the ware's **t2** [price threshold](./ware-prices/thresholds.md), `town + 0x4F0 + ware*0x10 + 0x8`. Skipped outright while `stock > t2`|`x1024`|
|ship weapon|`storage + 0x124 + i*4`|`REF[i] * ( ((-2 - town_index) & 3) + citizens/512 + 5 )`, with `REF` the u16 table at `0x00672CB4` - `1000, 1000, 2000, 2000, 2000, 1000`|`x512`|
|cutlasses|`storage + 0x2BC`|`markup + 5 * (snaikka_quality_level + 2)`|`x512`|

   The cutlass requirement is the one that reaches back into the shipyard block:
   `snaikka_quality_level` is the first byte of `town + 0x824`, and `markup` is
   `min(15, trunc(utilization_markup) / 4)` taken from the shipyard's `town + 0x818` when it
   is above zero, and `0` otherwise. So a busy shipyard building good snaikkas raises the
   number of cutlasses the town considers itself to need, from 10 at the floor to 40 at the
   ceiling.

   Note the scales differ: at the same fraction of its requirement a hand weapon scores twice
   as high as a ship weapon or cutlasses, and higher scores lose. Ship weapons and cutlasses
   are therefore systematically favoured over hand weapons of equal relative scarcity.

The producer then makes that one item at four times the base rate (`sar esi,0xe` rather than
`0x10`), consumes its inputs, scales the result down by whichever input is scarcest, and adds
it to the **stock**: `storage + 0x4 + ware*4` for a hand weapon (`0x0050FB16`),
`storage + 0x124` for a ship weapon (`0x0050FB48`), `storage + 0x2BC` for cutlasses.

### Eligibility Follows the Shipyard
One number gates both lists. `0x0052BB00` turns the town's **shipyard experience**
(`town + 0x810`) into a tier:

```
tier = min(5, shipyard_experience / 420000 + 2)
```

For **ship weapons** the tier is the loop bound outright: the scan runs indices `0..tier`
inclusive (`0x00510BDE`), so a weapon is eligible only once the tier reaches its index.

|tier|from experience|ship weapons eligible|highest hand weapon|
|-|-|-|-|
|2|0|small catapult, small ballista, large catapult|Bow|
|3|420,000|+ large ballista|Crossbow|
|4|840,000|+ bombard|Crossbow|
|5|1,260,000|+ **cannon**|Carbine|

Sword additionally requires the town to hold pig iron (`0x00510B1D`). Because the scan only
ever returns one item, a weapon outside the tier is never made at all: its stock stays at `0`
while its Production figure still reads its base rate every day.

That `town + 0x810` really is shipyard experience is worth stating, because gating *hand*
weapons on it looks odd. `0x004E21F6` folds the pending experience at `+0x814` into it inside
a routine that walks the town's shipbuilding queue from `+0x81C`, then clears the pending
figure. The shipyard tick feeds that pending figure with **the work it did this tick** - the
same amount it adds to the ship's progress at `ship + 0x18` (`0x00508D41` against
`0x00508CFE`). **Repairs count as much as new builds**: the tick's completion path adds the
work first, and only then tests `ship + 0x134 == 6` (`0x00508B86`) to pick the letter type the
owner receives - `7` when the status is `6`, `10` otherwise. Whichever way that test falls,
the experience is already banked, so a yard kept busy patching hulls raises the town's
armament tier exactly as one laying down new keels does. The same weaponsmith query reads two more fields of that block - the utilization
markup at `+0x818`, and the snaikka quality level at `+0x824`, which scales the cutlass
threshold. And the tier has five call sites in all, four of them feeding `0x0051A4E0`, the
routine that picks which ship weapon to fit to a ship (it refuses below tier 2 and splits
again above tier 4).

So this is not a weaponsmith quirk: P3 keeps a single **armament level** per town, derives it
from how much shipbuilding the town has done, and uses it for the smithy's hand weapons, the
smithy's ship weapons, and what gets mounted on ships.

### The Inputs Are Not the Same
All armaments draw on the same four wares - **timber, iron goods, leather and hemp** - but in
different proportions, and hand weapons are on a different scale entirely.

A hand weapon consumes, per unit produced, **10x timber** and **1x** each of iron goods,
leather and hemp.

A ship weapon consumes a **percentage** of its output, one byte per weapon in four tables:

|Ship weapon|timber `0x6735C4`|iron goods `0x6735D4`|leather `0x6735CC`|hemp `0x6735DC`|
|-|-|-|-|-|
|Small catapult|50%|10%|10%|10%|
|Small ballista|50%|10%|10%|10%|
|Large catapult|50%|10%|10%|10%|
|Large ballista|50%|10%|10%|10%|
|**Bombard**|**25%**|**30%**|**5%**|**5%**|
|**Cannon**|50%|**60%**|10%|10%|

The two gunpowder weapons are the outliers, and the cannon is doubly so: it is also the only
ship weapon that misses the `x2` output multiplier the other five get (`add eax,eax` at
`0x0050F7F3`, skipped by the `jae` above it), so it is
made at half their rate while needing six times their iron goods.

## Employees
`field_4_employees` is a real per-town workforce, and it is the term that makes `+0xC4`
staffing-dependent. `0x005101D0` computes a target per facility type and the tail at
`0x00510764` moves the count toward it; a facility at zero employees is skipped entirely
(except types `0x00` and `0x01`).

At world setup the counts come from a byte table at `0x006729D0`, indexed by type and scaled
by a **year ramp**: nothing below year 1300, a fixed maximum above 1400, linear in between
(the year is `[0x006DE4A2]`), divided by 3. Slots still at zero are skipped, so only
facilities the town actually has get staffed. Militia, Shipyard, Construction and
Weaponsmith are seeded directly with 10, 10, 5 and 2 at `0x00545CFC` onward.
