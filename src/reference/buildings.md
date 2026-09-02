# Buildings
New building ids, used in operations that create new buildings, are represented by the following enum:
```rust
pub enum NewBuildingId {
    Well = 0x28,
    Tower = 0x29, // Cannon, Bombard; Gate, Port
    HousePoor = 0x2a,
    PitchShoot = 0x2f,
    HouseWealthy = 0x50,
    HouseRich = 0x51,
    FarmGrain = 0x53,
    FarmHemp = 0x54,
    FarmSheep = 0x55,
    FarmCattle = 0x56,
    FishermansHouse = 0x57,
    Brewery = 0x58,
    Apiary = 0x59,
    WeavingMill = 0x5a,
    Workshop = 0x5b,
    Vineyard = 0x5c,
    HuntingLodge = 0x5d,
    Saltworks = 0x5e,
    IronSmelter = 0x60,
    Pitchmaker = 0x61,
    Brickworks = 0x62,
    Pottery = 0x63,
    Sawmill = 0x64,
    Hospital = 0x65,
    Warehouse = 0x66,
    Mint = 0x67,
    School = 0x68,
    Chapel = 0x69,
}
```

At `0x00672FC0` is a table that maps building ids to new building ids:

|BuildingId|NewBuildingId|
|-|-|
|0x00|0x2c|
|0x01|0x1b|
|0x02|0x0b|
|0x03|0x08|
|0x04|HuntingLodge|
|0x05|FishermansHouse|
|0x06|Brewery|
|0x07|Workshop|
|0x08|Apiary|
|0x09|FarmGrain|
|0x0a|FarmCattle|
|0x0b|Sawmill|
|0x0c|WeavingMill|
|0x0d|Saltworks|
|0x0e|IronSmelter|
|0x0f|FarmSheep|
|0x10|Vineyard|
|0x11|Pottery|
|0x12|Brickworks|
|0x13|Pitchmaker|
|0x14|FarmHemp|
|0x15|HouseRich|
|0x16|HouseRich|
|0x17|HouseRich|
|0x18|HouseWealthy|
|0x19|HouseWealthy|
|0x1a|HouseWealthy|
|0x1b|HousePoor|
|0x1c|HousePoor|
|0x1d|HousePoor|
|0x1e|Warehouse|
|0x1f|0x04|
|0x20|0x03|
|0x21|0x0b|
|0x22|0x07|
|0x23|0x00|
|0x24|0x02|
|0x25|0x05|
|0x26|0x09|
|0x27|0x33|
|0x28|0x52|
|0x29|Hospital|
|0x2a|Mint|
|0x2b|School|
|0x2c|Chapel|
|0x2d|0x37|
|0x2e|0x01|
|0x2f|0x5f|
|0x30|Tower|
|0x31|Tower|
|0x32|Tower|
|0x33|Tower|
|0x34|PitchShoot|
|0x35|PitchShoot|

## Town Structures
Ids `0x1E`..`0x2F` are the town's **unique structures** - one of each per town, and the
ones a player pays to have built:

|Id|Building|Id|Building|Id|Building|
|-|-|-|-|-|-|
|`0x1e`|Warehouse|`0x24`|Tavern|`0x2a`|Mint|
|`0x1f`|Church|`0x25`|Lender's House|`0x2b`|School|
|`0x20`|Town Hall|`0x26`|Public Bath|`0x2c`|Chapel|
|`0x21`|Armoury|`0x27`|Overland Trade Office|`0x2d`|Monument|
|`0x22`|Guild Hall|`0x28`|Well|`0x2e`|Shipyard|
|`0x23`|Market Hall|`0x29`|Hospital|`0x2f`|Mine|

Those names, and every other building id's, come from the **pointer table at
`0x006A57C8`**: 57 dwords indexed by building id `0x00`..`0x38`, each pointing into the
packed string block at `0x006A5688`. That is the id space the mask setter below bounds with
its `cmp al,0x30` (`0x0052190B`), and the construction pass bounds at `0x37`/`0x38`, so the
table is indexed exactly as the code indexes it - no alignment guess needed.

It names the low ids too, and **two of them are unique structures as well**, sitting outside
the `0x1E`..`0x2F` block:

|Id|Building|
|-|-|
|`0x01`|Repair Dock|
|`0x03`|Weaponsmith|

The remaining low ids are the many-per-town buildings - `0x04`..`0x14` the
[facilities](./facilities.md), `0x15`..`0x1D` the dwellings, `0x02` Road - and ids
`0x30`..`0x38` the defensive structures (towers, pitch shoots, wall and gates).

### The Built-Structures Mask
**`town + 0x76C` is a bitmask of the structures a town has**, one bit each. Every bit is
set by `add_town_building` at `0x00521900`, which dispatches on the building id through a
byte index table at `0x00522690` into the jump table at `0x0052262C` (ids `0x00`..`0x2F`).

Each case tests its own bit before setting it and returns 0 when it is already there,
which is what makes these buildings one-per-town, and most also require *other* bits first.
Every gate has the same shape, `(mask & (prerequisite | own bit)) == prerequisite`, so one
compare does both jobs:

|Bit|Building|Prerequisite|`or` site|
|-|-|-|-|
|`0x1`|Market Hall|- (own bit only, `0x00521EB1`)|`0x00521EC5`|
|`0x2`|Town Hall|`(mask & 0x3) == 0x1` - needs the Market Hall|`0x00521DAF`|
|`0x4`|**Weaponsmith**|`(mask & 0x6) == 0x2` - needs the Town Hall|`0x00521C51`|
|`0x8`|Armoury|`(mask & 0xC) == 0x4` - needs the Weaponsmith|`0x00521DD2`|
|`0x20`|Tavern|`(mask & 0x30) == 0x10`|`0x00522010`|
|`0x40`|**Repair Dock**|`(mask & 0x50) == 0x10`|`0x00521AD3`|
|`0x200`|**Church**|`(mask & 0x27F) == 0x7F` - all seven low bits|`0x00521D50`|
|`0x400`|**Mint**|`(mask & 0x600) == 0x200` - needs the Church|`0x0052217D`|
|`0x800`|Lender's House|`(mask & 0x810) == 0x10`|`0x00522036`|
|`0x1000`|**School**|`(mask & 0x1200) == 0x200` - needs the Church|`0x005221CD`|
|`0x2000`|Guild Hall|`(mask & 0x2010) == 0x10`|`0x00521E42`|
|`0x4000`|Public Bath|`(mask & 0x4010) == 0x10`|`0x00522093`|
|`0x10000`|Shipyard|`(mask & 0x10080) == 0x80`|`0x0052229A`|

Two quirks of the dispatch: ids `0x00` and `0x02` point straight at the shared `pop`/`ret`
refusal epilogue at `0x0052228E`, so the setter always rejects them, and the Shipyard's own
`or` sits *past* that epilogue, immediately before the store tail at `0x005222A3` - a
case-boundary scan loses it.

The **Repair Dock** is the bit that decides whether ships can be mended in a town: both
route-stop executors test it before ordering a repair and **clear the stop's R flag** when it
is absent (`0x00518993` for a lone ship, `0x005032EA` for a convoy). The repair
[operation](../towns/shipyard.md#ordering-a-repair) itself does not test it.

### Placed Is Not Finished
`0x00521900` runs when a site is **placed**, so `0x40` and `0x10000` mean a dock or a yard
has been paid for, not that it stands yet. Completion sets two further bits, from the
[construction](../towns/construction.md#what-completion-does) pass, for building ids `0x01`
and `0x2E` only:

|Bit|Meaning|Set at|
|-|-|-|
|`0x80`|a Repair Dock or Shipyard has **finished**|`0x0052012D` for a site with an owner, `0x0052072F` in the town-owned switch|
|`0x20000`|a **town-owned** Shipyard has finished|`0x0052036F`, `0x00520765`|

That is what the Shipyard's `(mask & 0x10080) == 0x80` asks for - a *finished* dock rather
than a placed one - and `0x80` is the bit
[`0x005101D0`](./facilities.md#producing-a-ware) tests to keep an unstaffed shipyard facility
running. The yard's ship-build list gates on `0x20000` instead (`0x0052B30A`), as does the
weekly shipyard task (`0x004E2283`).

Three bits remain unnamed. **`0x10`** has no setter anywhere in the executable, yet it gates
the Repair Dock, Tavern, Lender's House, Guild Hall and Public Bath and sits inside the
Church's `0x7F`, so it arrives with the town - through the savegame/scenario read at
`0x0051ECE0`. **`0x100`** is set at `0x004EA3E4`/`0x004EA440` and tested at `0x004EAE26` and
`0x0052E20B`. **`0x8000`** is set at `0x0041BDD8`/`0x0041BE3F`.

What the individual bits *do* is documented with the mechanic each affects: the Mint and the School
both act on [population](../towns/population.md) - the Mint on the
[rich divisor](../towns/population/levels.md), the School on the
[beggar intake](../towns/population.md#beggars) - and building any of Well, Hospital,
Chapel, Mint or School credits the builder's
[buildings reputation](../merchants/reputation.md#buildings).

**Well, Hospital and Chapel are not in this mask at all.** They are counted as bytes on the
town by the structure census `0x00520BE0` - Chapels into `town + 0x770`/`+0x771`, Hospitals into
`+0x772`/`+0x773`, Wells into `+0x789` - and those counts are what set a town's disaster odds:
hospitals and chapels against plague, wells against fire. See
[Plague and Fire](../towns/plague-and-fire.md).
