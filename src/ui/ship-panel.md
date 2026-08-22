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
|`+0xA0`|pointer to the current selection; its leading u16 is the selected ship index, and nothing after that is written|
|`+0xA00`|Auto trade view: stop row widget structs, stride `0xE8`; `row + 0x3E` is set while that row's stop is open in the goods dialog|

The selection pointer at `+0xA0` is what the panel's own code uses (`0x0048C363`, and
the route Load handler at `0x0048C92E` when filling `operations + 0x934`), making it a
reliable source for "which ship is selected" - it works on the world map and in town,
for own and foreign ships alike.

The field is cleared to `0` while nothing is selected, so it never points at the
previous selection. Opening any building window also clears it - the ship is visibly
deselected - and closing the window selects the same ship again, so the panel selection
cannot be read while a building window is on screen. The selection object itself is
heap-allocated and freed when the selection changes (`0x00487E74` frees the old one
before storing the new pointer), which is why a lingering copy of the pointer must not
be followed. Only its leading u16 carries meaning: re-selecting the same ship leaves
different values behind it, matching the high halves of neighbouring heap pointers, so
the following bytes are uninitialised rather than a type tag. The value behind it is always a **ship** index, never a convoy
one: selecting a convoy on the map reports the convoy's leader, and picking an
individual ship out of a convoy reports that ship. (A convoy itself is a `0x3C`-byte
record in the array at `ships + 0x08`; a ship names its convoy in `ship + 0x08` and the
members are chained through `ship + 0x06`.)

Two different buttons mean Goods, and only one of them opens a dialog. At the top of
the panel a barrel symbol switches the view, alongside Crew and Deck. Inside the
**Auto trade** view, every stop row carries a button labelled with the word "Goods",
and that one opens the [goods dialog](./auto-trade-goods-dialog.md) for the stop in
that row: it resolves the stop by walking the pool chain from `ship + 0x132` to the
first-stop marker and then forward by the row number (`0x0048C3A1`), and calls the
dialog's populate at `0x0048C432`.

Route edits made through the panel (town selection, "none", the active checkbox) are
operations: see [Set Trade Route Active](../operations/0068-set-trade-route-active.md)
and [Trade Route Stop Town Change](../operations/006a-trade-route-stop-town-change.md).
