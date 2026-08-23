# Auto Trade Goods Dialog
The "Automatic maritime trading in ..." dialog edits one stop of an applied trade
route. Its object is held in the static `0x006CBA74` (constructor `0x00403020`, vtable
`0x0066A7F0`, base class vtable `0x0066BC90`).

|Field|Meaning|
|-|-|
|`+0xA4`|pool index of the displayed stop; `-1` while the dialog is closed (the close method `0x004066F0` guards on it and stores `-1`)|
|`+0xA8`|ship index|
|`+0x4AE0`|per-ware order type, i32 per ware, **20 entries** (the dialog has no weapons rows): 0 unload, 1 sell, 2 load, 3 buy, 4 no order|
|`+0x4ADC`|ware index of the pending (uncommitted) edit, `-1` = none|
|`+0x547C`|flag byte, third argument of populate|

## Order Types
The order type is the dialog's own cache of what the stop record encodes in the signs of
price and amount; nothing outside the dialog reads it. `populate` derives it, splitting
the two office transfers by the amount's sign at `0x00405E30`:

|Mode|Order type|Record|
|-|-|-|
|0|unload ship -> office|price 0, amount < 0|
|1|sell to town|price > 0 (the minimum price)|
|2|load office -> ship|price 0, amount > 0|
|3|buy from town|price < 0 (the maximum price, negated)|
|4|no order|amount 0|

The row's single button cycles in one direction and wraps, verified in-game by click
count:

```
no order -> buy -> sell -> load -> unload -> no order
```

## Per-Ware Control Arrays
Ten pointer fields at `+0xF0` to `+0x114`, zeroed by the constructor (`0x00403148`), each
holding a `new[]` array of 20 objects of one widget class (constructor `0x004C6910`,
destructor `0x004C6A30`, element size `0xE8`) built through the array helper
`0x00639E9C`. Consecutive allocation makes the stored pointers come out evenly spaced by
`0x1238` (20 x `0xE8`, plus the array count header, rounded up), which is a handy sanity
check when reading them out of a live dialog.

Each array is one column of the row. Reading element 0's rectangle out of a live dialog
(the class is in the window family, so x is at `+0x14`, y `+0x18`, w `+0x2C`, h `+0x30`)
identifies them:

|Field|x|w|Column|
|-|-|-|-|
|`+0xF0`|0|96|ware-name button|
|`+0xF4` .. `+0x104`|562|48|the order-type button, five appearances sharing one rectangle|
|`+0x10C`|614|16|amount `-`|
|`+0x108`|708|16|amount `+`|
|`+0x114`|766|16|price `-`|
|`+0x110`|834|16|price `+`|

The order type selects among the middle five: `dialog + 0xF4 + mode*4` gives the array for
a ware's current type, and populate registers that one with the
[window manager](../ui.md#window-manager) (`0x004B4E30`, deregistering the previous with
`0x004B4EB0`). Because all five share a rectangle, they are five appearances of a single
control rather than five separate widgets - so the block is a sub-range of the ten, not a
self-contained table, and an order type of 5 would reach the amount `+` button.

Per-ware widget structs follow at stride `0x190`: `+0x8E0` holds the entered amount
(`-1` encodes Max, substituted with `1_000_000_000` on commit), `+0x2698` a text
buffer that is `atoi`'d and scaled by the barrel/bundle table at `0x00672C14`.

`populate` (`0x00405A20`, thiscall(this, stop_pool_index, ship_index, flag)) rebuilds
the whole dialog from the stop record. It is called by the [ship panel](./ship-panel.md)'s Goods button
(`0x0048C432`) and by the dialog's own stop-switching arrows, which follow the pool
chain from `+0xA4` (`0x004075E3` next, `0x0040763A` previous). The displayed texts are
sprintf-cached in the object, so in-place writes to the pool record stay invisible
until populate runs again.

## Deferred Commit and Undo
The +/- buttons do not write the stop record directly: they update the widget texts and
keep the edit pending (`+0x4ADC`). A commit helper around `0x00405461` builds
[operation 0x69](../operations/0069-route-stop-setting-change.md) from the per-ware
fields when the edit target changes, the stop is switched, or the dialog closes.

Undo therefore does not restore a snapshot - it discards the pending edits by
re-reading the pool record, which is why it reverts everything (order type included)
and why it does nothing after a stop switch: the switch committed.

On opening, the dialog normalizes the stop's inactive slots (amount 0, but possibly a
leftover base price) by enqueueing one operation 0x69 per such ware; these drain over
the following ticks.
