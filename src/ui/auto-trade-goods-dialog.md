# Auto Trade Goods Dialog
The "Automatic maritime trading in ..." dialog edits one stop of an applied trade
route. Its object is held in the static `0x006CBA74` (constructor `0x00403020`, vtable
`0x0066A7F0`, base class vtable `0x0066BC90`).

|Field|Meaning|
|-|-|
|`+0xA4`|pool index of the displayed stop; `-1` while the dialog is closed (the close method `0x004066F0` guards on it and stores `-1`)|
|`+0xA8`|ship index|
|`+0x4AE0`|per-ware mode array: 0 load, 1 sell, 3 buy, 4 none|
|`+0x4ADC`|ware index of the pending (uncommitted) edit, `-1` = none|
|`+0x547C`|flag byte, third argument of populate|

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
