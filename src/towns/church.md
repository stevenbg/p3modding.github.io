# Church

The town's church is the object **inline at `town + 0x794`**, initialised by `0x004FE2B0`:

|Field|Meaning|
|-|-|
|`+0x00`|jewellery-donation money|
|`+0x04`|money collected toward the current extension stage|
|`+0x08`|extension cooldown counter, initialised to `255`|
|`+0x0C`..`+0x0E`|units gathered of each extension material|
|`+0x0F`|extension stage, `0..3`|

Three operations reach it, dispatched inline in the operation switch at `0x00535CF0`ff:

|Op|Handler|What|
|-|-|-|
|`0x30`|`0x004FE557`|[feeding the poor](../merchants/reputation.md#feeding-the-poor)|
|`0x31`|`0x004FE420`|donation for the extension|
|`0x32`|`0x004FE2D0`|donation for jewellery|

All three credit the merchant's **social** reputation through the same routine
(`0x004F8AF0`) at `effective_amount * 0.0003 / (church_factor + 1) * base_rep_factor`
(the last term at `0x004F8B37`; see
[Recurring Constants](../merchants/reputation.md#recurring-constants) - it is 1.0 for
the player) - where
`effective_amount` is the part of the donation that fit under the cap below, not the part
that was paid, and `church_factor` is the [difficulty rank](../reference/game-settings.md),
which is only ever non-zero in multiplayer.

## Donations for jewellery (op `0x32`)

`0x004FE2D0` deducts the **whole donation** from the merchant first
(`0x004FE2EE`/`0x004FE2F4`) and only then applies the cap: money accumulates at
`church + 0x0` up to `12000 * (church_factor + 1)` (`0x004FE30A`-`0x004FE325`), with an
overshooting balance stored as `cap - 1`. **Gold past the cap is therefore lost** - it
buys neither decoration nor reputation.

The money **decays 100 gold per day** in the church's daily tick (`0x004FE885`).

The visible result is the **decoration level**,
`min(round(money / (2000 * (church_factor + 1))), 5)` (`0x004FE360`). The church window
consumes it only as `<= 4` versus `== 5`: the interior animation pair (intro/loop `7`/`6`
plain, `9`/`8` decorated, `0x005C958F`) and the caption string (`0x7956`/`0x7957`,
`0x005C98FA`). Since the level saturates at 4.5 steps and the cap is 6 steps, the last
stretch of money below the cap buys reputation and nothing visible.

## Donations for the extension (op `0x31`)

The stage at `church + 0xF` runs `0..3`; at `3` the church is fully extended. The current
stage's cost is

```
cost = BASE[stage] + church_factor * MULT[stage]
BASE = 20000 / 40000 / 60000     (u16 table at 0x006734B4)
MULT = 10000 / 15000 / 20000     (immediates in 0x004FE420 and 0x004FE390)
```

Donations are refused unless the cooldown counter at `+0x8` exceeds 100 (`0x004FE4C3`) and
the fund still lacks money. Like the jewellery, the money is taken **before** the fund is
clamped to the cost, so overshooting a stage wastes the difference.

### Materials, gathered daily from the town

A fully funded stage does not build by itself. The church's daily tick (`0x004FE877`)
gathers three wares out of the **town's own stock**:

|Material (ids at `0x006734BC`)|stage 0|stage 1|stage 2|
|-|-|-|-|
|Timber|20|20|30|
|Iron Goods|20|20|30|
|Bricks|50|50|60|

(amounts at `0x006734C0`, indexed `material * 4 + stage`). Per day and material it takes
`min(needed - held, (town_stock - t0_threshold/2) / scaling)` units - only what sits above
**half the ware's t0 price threshold** (`town + 0x4F0 + ware*0x10`, halved at
`0x004FE977`) - and subtracts the raw amount from the town stock (`0x004FEA28`). A town
short of bricks therefore stalls a fully funded extension.

Each completed stage also adds **+1 to every class's
[satisfaction](population/satisfaction.md#the-town-modifiers) base** - the extension's one
lasting effect beyond the visible building. The decoration level from jewellery donations
has no satisfaction effect: its only readers are the town scene and the church window's
artwork picks.

### Completion and the cooldown

Once the fund is full and all three materials are gathered, the tick (`0x004FEA65`ff)
zeroes the fund, the cooldown and the material counters, increments the stage, and calls
`0x00525390` on the town. Zeroing `+0x8` starts the **~100-day cooldown**: the tick
increments it daily and the donation gate is `> 100`. It is initialised to `255`, so a
town's first extension can be funded immediately.

The Extension tab's texts come from the status getter `0x004FEA97` (called from the church
window at `0x005CB56C` and the town view at `0x00599A57`): `5` = fully extended, `4` =
cooldown ("We are not currently planning any further extensions"), otherwise a funding
band against the stage cost.
