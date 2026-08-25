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

## Upgrade Levels
