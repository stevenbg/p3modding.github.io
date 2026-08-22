# Ship Panel
The scrollmap's right-side panel for the selected ship or convoy. Four buttons switch
its view - Goods, Crew, Deck and Auto trade - so the trade route is only one of the
things it shows, while the selection is common to all four. Vtable `0x0066F358`,
per-frame update `+0xF4` = `0x0048B3E0`.

Its static pointer is `0x006CE6D0`. The object is built once at startup like the
building windows, but its static sits apart from the `0x006E55xx`
[window cluster](../ui.md#window-objects), which is why it long looked as if it had
none: the mass-constructor allocates `0x83C8` bytes at `0x004266D2`, calls the
constructor (`0x00486D20`) at `0x004266E7`, and stores the result at `0x00426700`.
Reading the static is enough - `p3-api` exposes it as `UIShipPanelPtr`.

The object can also be captured by hooking the vtable slot at `0x0066F44C` (module
offset `0x26F44C`), which holds the update method; the hook receives the object as
`this` on every update. That was the original route and still works, but it is no
longer necessary.

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
