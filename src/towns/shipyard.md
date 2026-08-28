# Shipyard

## Shipbuilding
The `handle_build_ship` function is at `0x0052A360`.

### Build Capabilities
A shipyard's current quality level of each ship type is stored in the town's current *ship quality level* array, indexed by the ship type.
The `u8` values range from 0 to 3.

### HP and Capacity
A ship's HP and capacity depend on the respective quality level.
At `0x00673838` there is a table that holds the *structure base values* for every ship type and quality level:

|Type|Quality Level 0|Quality Level 1|Quality Level 2|Quality Level 3|
|-|-|-|-|-|
|Snaikka|15|19|23|25|
|Crayer|28|31|34|35|
|Cog|45|48|52|55|
|Hulk|55|59|65|70|

To the structure base value an unknown value is added, which appears to be always zero. The resulting structure value is capped at the value of QL 3.

HP and Capacity scale linearly with the structure value:
```
capacity = 2000 * structure_base_value
health = 2800 * structure_base_value
```
For a QL 3 hulk, this yields the expected capacity of 140.000 (700 barrels).


### Resources and Price
The `calculate_ship_build_cost` function is at `0x0052B2C0`.
The shipyard charges a *utilization markup* that increases when the shipyard is in use, and decreases if it is not.
There is one table per ship type, selected through the jump table at `0x0052B660` -
`0x0066DEB0` Snaikka, `0x0066DF10` Crayer, `0x0066DF70` Cog, `0x0066DFD0` Hulk - each three
rows of eight dwords, indexed by `min(quality, 2)` so QL 3 does not increase cost. The four
are contiguous, so they read as one twelve-row table.

The column order is fixed by `0x0052B381`..`0x0052B3A1`, which builds a five-entry array of
ware ids - `0x0B` timber, `0x08` cloth, `0x0C` iron goods, `0x11` hemp, `0x0F` pitch - and by
`0x0052B4AB`, which indexes the table with the **same counter** that indexes that array. So
column `i` is the requirement for ware `i` of that list. Requirements are in display units;
the divisor for raw units is the ware's 200 or 2000 from `0x00672C14` (`0x0052B48C`).

|Type|QL|Timber|Cloth|Iron Goods|Hemp|Pitch|Unknown|Base Price|Unknown|
|-|-|-|-|-|-|-|-|-|-|
|Snaikka|0|7|3|3|3|20|17|7,650|11,414|
|Snaikka|1|9|3|3|3|20|20|8,200|12,074|
|Snaikka|2|11|3|3|3|20|24|8,800|12,784|
|Crayer|0|12|5|5|5|30|29|18,270|24,450|
|Crayer|1|14|5|5|5|30|32|18,720|25,010|
|Crayer|2|16|5|5|5|30|34|19,890|26,290|
|Cog|0|18|3|4|4|40|46|16,560|22,296|
|Cog|1|20|3|4|4|40|50|16,500|22,346|
|Cog|2|22|3|4|4|40|53|17,490|23,446|
|Hulk|0|30|10|10|8|50|58|22,968|34,442|
|Hulk|1|33|10|10|8|50|64|23,040|34,679|
|Hulk|2|36|10|10|8|50|69|24,840|36,644|

### Where the Materials Come From
The build dialog's four columns are all filled by `calculate_ship_build_cost`, and each has a
different source. **"In stock" is only what the ordering merchant already owns** - the town
market is not part of it:

- his **office** in that town, resolved through `0x005308A0(merchant, town)`, summing
  `office + 0x4 + ware*4` (`0x0052B3BD`);
- the cargo of **his own ships docked there**: it walks his ship chain from `merchant + 0xE`
  and, for each ship whose `+0x39` is this town, whose status (`+0x134`) is below `4` and whose
  `+0x136` bit `0` is clear, adds `ship + 0x54 + ware*4` (`0x0052B444`).

The town's own stock enters one step later, as the bound on what can be bought
(`0x0052B4A2`..`0x0052B4FF`, per ware, everything in display units):

```
owned = (office + own docked ships) / scale        ; the "in stock" column
if owned >= required:
    buy = 0; cost = 0
else:
    short = required - owned
    cost  = get_buy_price(ware, town, short * scale)    ; 0x0052B4E0
    if short <= town_stock / scale:
        buy = short
    else:
        buy = town_stock / scale                        ; capped
        can_build = false                               ; 0x0052B4FA
```

A ware the town cannot cover clears the build flag for the whole order, so one missing
material blocks the hull regardless of the other four.

Note the ordering: `get_buy_price` runs at `0x0052B4E0` on the shortfall *before* the cap is
applied at `0x0052B4FF`, so on the capped branch the cost is computed for more than will be
bought. Whether the confirmed order charges that figure has not been checked.

The ship price is calculated as follows:

```python
def structure_markup(structure):
    if structure <= 20:
        return 900
    if structure <= 30:
        return 840
    if structure <= 40:
        return 780
    if structure <= 50:
        return 720
    if structure <= 60:
        return 660
    else:
        return 600

price = base_price
    + structure_markup(structure_base_value)
    + utilization_markup
    + resource_prices
```

## Repairs
Repairing is driven by a queue on the shipyard, but - unlike shipbuilding - **the ships in
that queue do not compete with each other**. Each of them is mended at the yard's full rate,
so ten ships under repair finish just as fast as one.

### Ordering a Repair
Operation `0x03` reaches `handle_repair_ship` at `0x0052ACD0`, a `thiscall(town, ship*)`. It
refuses unless the ship's status (`+0x134`) is below `4`, i.e. it is in port.

The cost is per ship, from `0x0052A9E0`, which reads the shipyard's *utilization markup* and
steps off it in bands. If the ship belongs to a convoy the handler sums the cost over every
member, walking the convoy's member chain from `convoy + 0x0A` through `ship + 0x6`, and
likewise sums the members' `+0x18` health against their `+0x14` maximum to decide whether
there is anything to repair at all.

Three outcomes, each announced by a letter (the [message pool](../letters.md) entry's type
byte at `+0x4`):

|Condition|Type|Text|
|-|-|-|
|no damage found|-|*"%s %s cannot be repaired. The shipyard in %s could not find any damage..."*|
|money short|`2`|*"...you cannot pay the agreed price of %i..."*|
|accepted|`1`|*"The shipyard in %s is repairing %s for you. The actual condition amounts to %i%%..."*|

On acceptance the **full price is deducted immediately** (`0x0052AF68`), booked to the
merchant's expense field `+0x4AC`, and the ship's status becomes `4`. The condition printed
in the letter is `health * 100 / max_health` (`0x0052AF34`).

### The Repair Queue
The shipyard keeps two independent ship chains, both linked through `ship + 0x6` - the same
multi-purpose link the ship tick lists and the convoy member chain use:

|Field|Chain|Ship status|Appended by|Worked by|
|-|-|-|-|-|
|`town + 0x81C`|under construction|`0x0E`|`0x00507EA4`|`0x005083B0`|
|`town + 0x81E`|under repair|`6`|`0x0052AC80`|`0x00508AA0`|

A ship reaches the repair chain on the first ship tick after the order: the status-`4`
handler `0x00506A19` splices it out of the in-port list, writes `0xFFFF` into its `+0x6`,
sets status `6`, and calls `0x0052AC80` to append it at the tail. From then on the ship is in
no tick list at all - the shipyard is the only thing that touches it.

If the ship's trade route was active (`+0x136 == 1`) the flag is parked as `2` for the
duration, and restored to `1` on completion (`0x00508BF6`).

### Repair Progress
A town's facilities tick when [the town does](../time.md#towns), once per day. The shipyard
branch of the per-facility production step (`0x005101D0`, branch `0x0051025A`) calls
`0x0050E570`, which computes one work amount and hands the **same** amount to both chains:

```python
work = facility.employees * 100          # 0x0050E5D8
advance_construction(town, work)         # 0x005083B0
advance_repairs(town, work)              # 0x00508AA0
```

A [fully staffed shipyard](../reference/facilities.md#full-workforce-per-type) has 40
workers, so a yard at full staffing does **4000 hull points per day**. Since a ship's maximum
health is `2800 * structure_base_value`, that is a little over 2% of a quality-level-3 hulk
per day.

`advance_repairs` walks the whole chain from `town + 0x81E`:

```python
work = args.work                     # re-read every iteration, 0x00508ADE / 0x00508C5F
for ship in repair_chain:            # 0x00508AD9 .. 0x00508C80
    ship.health += work              # 0x00508AF8, ship + 0x18
    town.pending_experience += work  # 0x00508C6F, town + 0x814
    if ship.health >= ship.max_health:
        finish(ship)
```

The work amount is loaded from its stack slot on every pass of the loop and is **never
divided by the queue length nor decremented as ships consume it**. That is what makes repairs
non-competing, and it is the one place where repair and construction differ in kind:
construction really does queue, which is why the shipyard window's estimate `0x0052AFC0` sums
the outstanding work of every ship *ahead* of the one asked about.

### Completion
When a ship reaches its maximum health it is clamped to it, unlinked from the chain, relinked
into the in-port list at `ships + 0xE8`, and given status `5` (`0x00508BCE`). Only the part of
the last work amount that was actually used is credited as experience (`0x00508B30`).

A letter reports it - type `7`, *"Repairs completed in %s"*, chosen at `0x00508B86` by testing
whether the status was `6` (repair) or not (a newly built ship, type `10`). Ships on a trade
route are skipped: the letter is suppressed when `ship + 0x136 & 3` is set (`0x00508BA1`).

**Switching a ship's trade route back on ends its repair.** A queued ship whose `+0x136`
reads exactly `1` is taken out of the chain unfinished and put back with status `5`
(`0x00508C11`). Since entering the yard parks an active flag as `2`, this happens only when
the route is re-activated while the ship is being repaired - the case behind the message
*"Repairs on the ship will be discontinued on sailing."*

### Repairs and the Utilization Markup
Every ship in the repair chain credits its full work amount to the town's *pending
experience* separately, so N ships being repaired earn the yard N times the experience.
[Scheduled task `0x06`](../scheduled-tasks/0006-update-shipyard-experience.md) banks that
weekly - and because pending experience is also a term in the utilization markup, a yard kept
busy with repairs becomes both more experienced and more expensive.

## Upgrade Levels
