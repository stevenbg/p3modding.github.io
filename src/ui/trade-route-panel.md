# Trade Route Panel
The scrollmap's right-side panel showing the selected ship or convoy and its trade
route. The object has no static pointer; it can be captured by hooking the vtable slot
at `0x0066F44C` (module offset `0x26F44C`), which holds the panel's per-frame update
method `0x0048B3E0` - the hook receives the object as `this` on every update.

|Field|Meaning|
|-|-|
|`+0xA0`|pointer to the current selection; the selection's first u16 is the selected ship index|
|`+0xA00`|stop row widget structs, stride `0xE8`; `row + 0x3E` is set while that row's stop is open in the goods dialog via its Goods button|

The selection pointer at `+0xA0` is what the panel's own code uses (`0x0048C363`, and
the route Load handler at `0x0048C92E` when filling `operations + 0x934`), making it a
reliable source for "which ship is selected" - it works on the world map and in town,
for own and foreign ships alike.

The panel's Goods button computes the clicked row's stop by walking the pool chain
from `ship + 0x132` to the first-stop marker and forward by the row number
(`0x0048C3A1`), then calls the
[goods dialog](./auto-trade-goods-dialog.md)'s populate (`0x0048C432`).

Route edits made through the panel (town selection, "none", the active checkbox) are
operations: see [Set Trade Route Active](../operations/0068-set-trade-route-active.md)
and [Trade Route Stop Town Change](../operations/006a-trade-route-stop-town-change.md).
