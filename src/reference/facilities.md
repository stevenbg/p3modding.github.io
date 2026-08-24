# Facilities
Business and miscellaneous buildings are grouped into *facilities*. Every town holds a
**21-slot facility array at `town + 0x840`** (stride `0x10`), and it is indexed **by facility
type**: the constructor `0x005100F0` writes the slot index into `field_6_type`, so slot `n`
is always type `n`. Types `0x00`..`0x14` therefore cover the whole array. What each one
produces, and how much, is in [Production](../towns/production.md).

Types `0x00`..`0x03` are the municipal facilities and exist in every town; the rest are the
ware producers and vary by town.

```rust
pub enum FacilityId {
    Militia = 0x00,
    Shipyard = 0x01,
    Construction = 0x02,
    Weaponsmith = 0x03,
    HuntingLodge = 0x04,
    FishermansHut = 0x05,
    Brewery = 0x06,
    Workshop = 0x07,
    Apiary = 0x08,
    FarmGrain = 0x09,
    FarmCattle = 0x0a,
    Sawmill = 0x0b,
    WeavingMill = 0x0c,
    Saltworks = 0x0d,
    IronSmelter = 0x0e,
    FarmSheep = 0x0f,
    Vineyard = 0x10,
    Pottery = 0x11,
    Brickworks = 0x12,
    Pitchmaker = 0x13,
    FarmHemp = 0x14,
}
```
Type `0x02` produces no ware: it is the town's building workforce, and its employee count
gates the [construction](../towns/construction.md) queue.

## Which Ware a Facility Produces
Two byte tables map between the two, and neither is a straight inverse of the other because
three wares break the one-to-one rule.

`0x00672C2C`, indexed by **facility type**, gives the type's *primary* ware. Only entries
`0x03`..`0x14` are meaningful - the first three read `0x06` as filler - and it is what
`0x0051014C` uses to find the ware whose [price thresholds](../towns/ware-prices/thresholds.md)
decide whether a facility keeps running:

|Type|Ware|Type|Ware|Type|Ware|
|-|-|-|-|-|-|
|Weaponsmith|Sword|FarmGrain|Grain|IronSmelter|PigIron|
|HuntingLodge|Skins|FarmCattle|Meat|FarmSheep|Wool|
|FishermansHut|Fish|Sawmill|Timber|Vineyard|Wine|
|Brewery|Beer|WeavingMill|Cloth|Pottery|Pottery|
|Workshop|IronGoods|Saltworks|Salt|Brickworks|Bricks|
|Apiary|Honey|Pitchmaker|Pitch|FarmHemp|Hemp|

`0x00672C88`, 24 bytes indexed by **ware**, gives the type that produces it. It is what the
town information window's two produced-ware lists walk (`0x005B7DAA` for the effective list,
`0x005B7EFD` for the low one), skipping any entry `<= 3`:

```
09 0a 05 06 0d 08 00 10 0c 04 ff 0b 07 0a 0f 13 0e 14 11 12 03 03 03 03
```

Three entries are not a plain inverse:

|Ware|Entry|Why|
|-|-|-|
|Spices|`0x00` Militia|nothing produces spices; every reader skips types `<= 3`|
|Leather|`0x0a` FarmCattle|shared with meat - the cattle farm's second output|
|**WhaleOil**|**`0xFF`**|**no facility at all** - see [Production](../towns/production.md#whale-oil-has-no-facility)|

Whale oil is the only ware carrying the `0xFF` sentinel, and the only one whose productivity
lives outside this array, in `town + 0x2CC`.

## Full Workforce Per Type
`0x006735E8`, a u8 table of 21 indexed by facility type, is the workforce a facility is
credited with when it is fully staffed:

|Type|Workers|Type|Workers|Type|Workers|
|-|-|-|-|-|-|
|Militia|250|Apiary|60|IronSmelter|70|
|Shipyard|40|FarmGrain|68|FarmSheep|50|
|Construction|25|FarmCattle|60|Vineyard|93|
|Weaponsmith|5|Sawmill|48|Pottery|58|
|HuntingLodge|72|WeavingMill|60|Brickworks|27|
|FishermansHut|65|Saltworks|54|Pitchmaker|26|
|Brewery|68|Workshop|78|FarmHemp|62|

It caps the worker target at `0x005101BF`, and it is the term the **nominal** production
figure at `town + 0x490` uses where the actual figure uses `field_4_employees`. The 21 values
are constant-folded into the dispatch branches described below rather than read from the
table at production time - `push 0x3c` for the apiary at `0x00510415`. FishermansHut is the
one type computed at runtime: 65, or 72 in a whaling town (`0x0051032C`).

Brickworks at 27 and Pitchmaker at 26 are the two small types, which is the same split that
makes a *merchant's* building of those types hold 15 workers rather than 30.

## Producing a Ware
`0x005101D0` is the per-facility production step, called by the town tick and by world
setup. It gates on staffing first: employees `> 0` proceeds, and at zero only Militia
continues unconditionally, with Shipyard continuing only when `town + 0x76C & 0x80`. It then
dispatches on `field_6_type` through a **21-entry jump table at `0x00510880`** to one bespoke
routine per type, each with its own hardcoded wares and scale factors:

|Type|Branch|Producer|Type|Branch|Producer|
|-|-|-|-|-|-|
|Militia|`0x00510234`|`0x00510C50`|Sawmill|`0x005104D5`|`0x0050ECD0`|
|Shipyard|`0x0051025A`||WeavingMill|`0x00510515`|`0x0050EEB0`|
|Construction|`0x0051029F`||Saltworks|`0x00510555`|`0x0050EF50`|
|Weaponsmith|`0x005102C2`|`0x00510AA0`|IronSmelter|`0x0051059A`|`0x0050F650`|
|HuntingLodge|`0x005102EC`|`0x0050ED70`|FarmSheep|`0x005105DA`|`0x0050F060`|
|FishermansHut|`0x0051032C`|`0x0050E690`|Vineyard|`0x0051061A`|`0x0050F100`|
|Brewery|`0x00510381`|`0x0050E8B0`|Pottery|`0x0051065A`|`0x0050F220`|
|Workshop|`0x005103C6`|`0x0050FBC0`|Brickworks|`0x0051069F`|`0x0050F2E0`|
|Apiary|`0x00510406`|`0x0050EA00`|Pitchmaker|`0x005106DC`|`0x0050F390`|
|FarmGrain|`0x00510446`|`0x0050EAD0`|FarmHemp|`0x0051071E`|`0x0050EBF0`|
|FarmCattle|`0x00510486`|`0x0050F460`||||

Each producer takes `(storage, full_workforce, flag)` and does the same two things: add
`efficiency * full_workforce / k` to the nominal array at `town + 0x490`, and
`employees * efficiency / k` to the actual array at `storage + 0xC4`, alongside the stock at
`storage + 0x4` and an 8-slot history ring at `storage + 0x13C` (24 wares, stride `0x10`,
indexed by `([0x006DE4B4] >> 8) & 7`). A two-output type does it twice, and the `flag`
argument gates the second ware.

The town's facilities are stored within the town struct.
```
00000000 struct facility // sizeof=0x10
00000000 {                                       // XREF: town/r
00000000     int field_0_efficiency;             // per-town, per-facility savegame state; the number the output formula uses. World setup SEEDS it as BASE_EFFICIENCY[type] * field_8 / 1024 at 0x00545E48, but a loaded save may hold anything - read it, do not compute it. See Towns > Production
00000004     unsigned __int16 field_4_employees;
00000006     unsigned __int8 field_6_type;       // always equals the slot index
00000007     unsigned __int8 field_7_town_index;
00000008     __int16 field_8_productivity;       // 1024 effective, 768 or 683 low, 0 = no such facility here. Written by 0x00545912 from the scenario's two ware bitmaps, and the field the town information window's produced-ware lists read - never field_0. Whaling has no slot: its grade is in town+0x2CC
0000000A     __int16 field_A;                    // posts this facility may fill but has not yet: the employment tail 0x00510787 drains it into field_4_employees as the target rises. Counted alongside employees by the population target, so a town's population does not dip mid-hire. Non-zero in only 22 of 510 measured slots
0000000C     __int16 field_C;
0000000E     unsigned __int16 field_E;
00000010 };
```
