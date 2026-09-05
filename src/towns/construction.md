# Construction
Buildings do not appear the moment they are paid for. A town keeps a list of pending
construction sites, and one of its [facilities](../reference/facilities.md) - type `0x02`,
the town's building workforce - advances them.

## The Site List
The array hangs off the town, stride 8 bytes per site:

|Field|Meaning|
|-|-|
|`town + 0x75C`|array base pointer|
|`town + 0x760`|freelist head - finished sites are pushed back here|
|`town + 0x762`|u16 head of the **pending chain**: the sites still being built, linked through their `+0x4`, ended by an index at or past the bound. The daily pass walks exactly this chain|
|`town + 0x764`, `town + 0x768`|chains of completed town-owned structures|
|`town + 0x776`|u16 array bound|
|`town + 0x7AC`|map stride, used as `x * stride + y`|
|`town + 0x80C`|map bytes, tested against `0x80`|

A site is:

|Offset|Meaning|
|-|-|
|`+0x0`|map x (set to `0xFF` when the slot is freed)|
|`+0x1`|map y|
|`+0x2`|owning merchant index; a value at or above the merchant count `[0x006DE4AA]` means the town owns it|
|`+0x3`|[building id](../reference/buildings.md), `1`..`0x38`|
|`+0x4`|u16 next-site index - chain or freelist link|
|`+0x6`|**work left**, in workforce units. Placement writes the building's total from the byte table at `0x00672BB4` (indexed by building id, `0x00523CC5`); the daily pass pays it down and completes the site when the remainder fits (below). The pass hands the map renderer `total - left` and `total` for the site's progress picture (`0x005204B7`)|

## The Workforce Is a Daily Budget
Facility type `0x02` exists in all 24 towns and produces no ware. Its only job is this list:
its employee count is passed to `0x0051FF30`, the sole routine dedicated to it, which walks
the sites and advances each one it can afford:

```
budget = employees                      ; 0x0051FFA0, loaded once
for each site in the pending chain:
    if site.left <= 5 and site.left <= budget:
        complete the site               ; unlink, left = 0, budget -= left, dispatch by id
    else:
        pay = min(budget, 5)            ; 0x00520479..0x00520493
        site.left -= pay
        budget -= pay
    stop when budget is 0
```

The employee count is **spent**, not compared: `ebx` is loaded once before the walk and never
reloaded inside it, so one tick pays into as many sites as the budget covers, five units at
most per site per day. A 25-worker town works five sites a day; the sites it could not
afford wait for tomorrow, in chain order.

## Remaining Building Time
The building info panels print the game's own estimate, `0x0051D6F0(town, site_index) ->
i32` (thiscall, `ret 4`; callers `0x005A881B`, `0x005B0234`, `0x00500A3E`). It walks the
pending chain to the site, letting each site ahead take `min(budget, 5)` of today's budget
(`town + 0x864`, the workforce facility's employees). If the budget is gone before the site
it returns a **negative** number: `-1` for the first unfunded site, `-n` for the n-th; the
panel prints "Building will start soon" for `-1`, "This building is in position n. in the
order for completion" for the rest (`0x005A0FBA` negates it), and nothing at or below
`-1000`. Otherwise it returns `ceil(left / min(budget, 5))`, then `0x0052DFF0(town_index,
days)` subtracts one if this town's daily pass has already run today: towns are processed
at staggered ticks, `town_index * 8 + 7` of the 256-tick day (`- 253` from `0x100` up),
compared against the time of day `[0x006DE4B4] & 0xFF`. A site with no work left returns
`0`, an index past the bound `0x10000000`. Site records resolve through `0x0051F1F0(town,
index)` = `[town + 0x75C] + 8 * index`, 0 past the bound. While town flag `+0x2C8 & 0x10` is
set (the bit that also suspends [imports](./production.md#imports-from-outside-the-hanse)),
a site whose id is above `0x30` (`0x31` for one merely ahead in the chain) or whose map tile
`[town + 0x80C][x * [town + 0x7AC] + y]` is below `0x80` is passed over without taking
budget, and asked about it the routine returns `-1` outright (`0x0051D76A`..`0x0051D808`).

With no employees the routine returns immediately and nothing is built at all.

The employee target is unusual in that it is the only one driven by population rather than
production (`0x0050E610`):

```
base = 5 * (total_citizens / 5000 + 5)
sum  = satisfaction[poor]*5 + satisfaction[wealthy]*2 + satisfaction[rich]
if sum >= 0:  target = base
else:         target = max(5, base + (sum + 7)/8)   # truncated toward zero
```

Note the `jns` at `0x0050E66E` skips the entire satisfaction block, not merely the
negative-rounding correction: **satisfaction is a penalty and never a bonus**, and the
minimum of 5 only exists on that path. The three satisfaction values are read with `movsx`,
so they are signed; beggars at `+0x306` are not read at all.

In a town below 5000 citizens with non-negative satisfaction the target is therefore
`5 * 5 = 25`, which is what every town in a measured save shows. Because the budget is spent
per site, 25 workers is what limits **how many** sites move on a given day, so a town with a
long site list feels it whether or not its citizens are content - and a satisfaction penalty,
which can take the target down to 5, cuts that throughput by four fifths.

## What Completion Does
The owner is sent a **type `3` message** through `add_message` (`0x004D6530` on the letter
pool `0x006DD730`) carrying the town index and the building id, and the finished building is
linked into a chain according to its building id (switch at `0x005200BB`, index
`building_id - 1`, byte table `0x00520534`, jump table `0x0052050C`):

|Building ids|Linked into|
|-|-|
|`0x04`..`0x14`, `0x1E`|`office + 0x2CE` - the 17 production facilities plus the warehouse. This chains the finished **site records**, one per physical building, through their `+0x4`. It is not the merchant's [production record chain](../merchants/trading-office.md) at `office + 0x2CC`, which holds one aggregate per facility type|
|`0x15`..`0x1D`|`office + 0x2D0` - the nine dwelling types, chained the same way|
|`0x03`, `0x1F`..`0x26`, `0x2D`, `0x2F`|handed to the town (`+0x2` set to `0xFF`) and chained at `town + 0x764`|
|`0x01`, `0x2E`|`town + 0x2D4` total citizens **`+= 4`** and `town + 0x2E0` poor citizens `+= 4`, plus `town + 0x76C |= 0x80` and the Shipyard facility's employees set to `1`|
|`0x34`, `0x35`|paired with a matching site of id `0x37`/`0x38` at the same coordinates, rewriting it|
|`0x27`..`0x2C`|chained at `town + 0x764` like the row above, but **without** setting `+0x2` to `0xFF` - the site keeps its owner (`0x00520195`)|
|`0x30`..`0x33`|`+0x2` set to `0xFF` and chained at `town + 0x768` (`0x005201AC`)|
|`0x36`|the same, and **`town + 0x786` += 1** (`0x005201C7`)|
|`0x37`, `0x38`|the same, and **`town + 0x786` += 3** (`0x005201E9`)|
|`0x02`|nothing - its table entry shares the out-of-range target|

`town + 0x786` is what those last two rows make interesting: it is read by
`update_citizen_satisfaction`, where one of the nine modifiers divides by it
(`0x0051CA5C`), and a value of `0` gates a call to `0x00524C30`. Cross-referenced with the
[building ids](../reference/buildings.md), `0x30`..`0x33` are towers and `0x34`/`0x35` pitch
shoots, so these are the town's defensive structures and finishing one feeds citizen
[satisfaction](./population/satisfaction.md).

A second switch at `0x005202F9` handles town-owned sites, setting town flags (`0x80`,
`0x20000`) and either chaining the result at `town + 0x764`/`+0x768` or freeing the site to
the `town + 0x760` freelist.

Completing a dwelling does **not** change the citizen count here - it only links into
`office + 0x2D0`. The only direct population increase on this path is the `+= 4` for building
ids `0x01` and `0x2E`.
