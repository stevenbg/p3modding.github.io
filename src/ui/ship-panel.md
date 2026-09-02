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
|`+0x14`, `+0x18`|the panel's origin, **relative to its parent** - see [Drawing Into It](#drawing-into-it)|
|`+0x2C`, `+0x30`|the panel's width and height; measured `260 x 247`|
|`+0xA0`|pointer to the current selection; its leading u16 is the selected ship index, and nothing after that is written|
|`+0xB4`|a selection-kind state, with its own ids in `+0xB8`..`+0xC8` (`0`..`4`). Observed `1` for a single ship and `2` for a convoy; not otherwise established. The view switcher writes it together with `+0xCC`, which is why the two are easy to confuse|
|`+0xCC`|**the current view** - see below|
|`+0xA00`|Auto trade view: stop row widget structs, stride `0xE8`; `row + 0x3E` is set while that row's stop is open in the goods dialog|

## Which View Is Showing

`panel + 0xCC` carries the view, and the panel keeps the possible values in **fields**
rather than as immediates: the constructor writes `0`..`6` into `+0xD0`..`+0xE8`
(`0x00486D8D`..`0x00486DB3`). So every test in the executable reads
`cmp [esi+0xCC],[esi+0xE4]` and **searching the disassembly for a literal finds nothing**.

Measured by pressing each button:

|Value|View|
|-|-|
|`0`|Goods|
|`1`|Crew|
|`2`|Deck|
|`5`|**Auto trade**|

`3`, `4` and `6` exist as ids but no button produces them. The Auto trade value is
corroborated in code: `0x0048BEA7` loads `+0xE4` (= 5) and `0x0048BEAF` stores it into
`+0xCC` - that button's own handler - and the draw method's auto-trade branch compares
against `+0xE4` at `0x0048B1AF`. The switcher that writes the field is `0x00488150`.

## Drawing Into It

The draw method is vtable `+0x9C` = `0x0048B060`, thiscall with **four** stack arguments
(`ret 0x10`). A mod can hook the vtable slot, call through, and draw afterwards so its
output lands on top of the panel's own art.

Two things decide whether anything appears:

- **`+0x14`/`+0x18` are not the screen position.** The draw method *adds its second and
  third stack arguments* to them before using the result (`0x0048B07D`, `0x0048B07F`), so
  the origin is `arg2 + panel[0x14]`, `arg3 + panel[0x18]`. Using `+0x14` alone puts the
  output somewhere else entirely.
- **The panel clips to its own rect.** A coordinate outside `260 x 247` draws *nothing* -
  it does not spill onto the map - so an overlay that silently fails to appear is usually
  outside the rect rather than mis-hooked.

Panel-relative coordinates hold across resolutions: the same offsets land correctly at
800x600 and above, because the origin moves with the panel.

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
the following bytes are uninitialised rather than a type tag.

The value is always a **ship** index, never a convoy one - but it is **not** necessarily
the convoy's leader. Measured against a 101/161/180/187 convoy whose leader is 161,
both selecting the convoy as a whole and clicking one of its members reported ship
**101**; the panel gives back a member either way. A mod that needs the leader must
resolve it itself, through `convoy + 0x10`. Whether "the convoy as a whole" always
yields the lowest member index or the game's own member order is untested. (A convoy is
a `0x3C`-byte record in the array at `ships + 0x08`, and a ship names its convoy in
`ship + 0x08`. Membership is **not** the `ship + 0x06` chain, which is the ships-tick
list link and read `0xFFFF` on the measured leader.)

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
