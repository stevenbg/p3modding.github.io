# Scrollbar

The game has one scrollbar class, `CP2Scrollbar`, and a **list controller** that wraps it
and turns its position into a first visible row. The ship overview drives two of them;
the [auto trade window](./auto-trade-window.md) three. `p3-api`'s `ui::scroll_list`
wraps the pair, and `mod-tavern-details` puts one on its captains table.

## `CP2Scrollbar`

Constructor `0x00460870` (thiscall, no arguments; a twin taking one follows it), vtable
`0x0066E120`, about `0x380` bytes. It constructs the widget base with the name at
`[0x006985B4]`, then its three child buttons - `+0x94` up, `+0x17C` down (`0x004C6910`),
`+0x264` thumb (`0x0046CC20`) - then writes its vtable and resets. **`0x00460A70` is the
destructor**, not a constructor: vtable slot 0 (`0x004620E0`) calls it, its exception
states count down, and it ends in the base destructor chain.

|Field|Meaning|
|-|-|
|`+0x360`|scrollable flag: the total exceeds the track|
|`+0x364`|content total, in pixels|
|`+0x368`|track length minus the buttons|
|`+0x36C`|**position**, in pixels|
|`+0x370`|track length (the area's height)|
|`+0x374`|step (the row height)|
|`+0x378`|last reported position|
|`+0x37C`|**range**: track / step = visible rows|
|`+0x380`..`+0x390`|the **wheel catchment** `RECT`|

|Function|Signature|What|
|-|-|-|
|vtable `+0x8` = `0x00460B70`|thiscall(CString by value, id), `ret 8`|load `[P2Scrollbar<id>]` from the ini: `UpButtonID`, `DownButtonID`, `ScrollButtonID`, `BackgroundTexID`, `Files`. `parchment.ini` (inside `p2arch0_eng.cpr`) holds ids 0 and 1, both pointing their buttons at `[Button0]`/`[Button1]`/`[Button2]` of the same file. The string is destroyed by the callee|
|`0x00460EF0`|thiscall(area*, total, step, container), `ret 0x10`|**place**: the bar goes against the area's right edge (`+0x14 = right - button width`, `0x00461108`), the area's height is the track and `+0x37C = height / step`, position 0, `+0x364 = total`, and the area is copied to `+0x380`. Then it removes and re-adds **itself and its three buttons** to `container` - the top scene when 0 - and puts itself in the container's scrollbar list `+0x144` (`0x004B50F0`, add once)|
|`0x00461480`|thiscall()|**detach**: remove the three buttons and the bar from the top scene|
|`0x00461FD0`|thiscall()|release the thumb (its vtable `+0x44`); the ship overview calls it before re-placing|
|vtable `+0xCC` = `0x00461EF0`|thiscall(bool)|show/hide the bar and its buttons; sets `+0x4C` to `(0, 0, w, h)`|
|vtable `+0xF4` = `0x00461590`|thiscall()|per-frame update: thumb drag, moving `+0x36C` by `+0x374`|
|vtable `+0x9C` = `0x00460DA0`|thiscall(context, x, y, z)|own draw: track and buttons|
|`0x00461FE0`|thiscall(context, x, y, z), `ret 0x10`|composite draw for parents that do not register the bar as a container child (the auto trade window, `0x0048AE76`..)|
|`0x00461320`|thiscall(pixels), `ret 4`|scroll by; gated on `+0x360`|
|`0x00461450`|thiscall(out*) -> bool|read the position into `*out`; true when it differs from `+0x378`|
|`0x00461CA0`|thiscall(total)|set the content total; maintains `+0x360`|

The area handed to `0x00460EF0` is therefore **the whole scrollable list**, not the bar
column: the bar places itself at its right edge, and the mouse wheel works everywhere
inside it.

### The wheel

`WM_MOUSEWHEEL` reaches `0x004C0FA0` through the MFC message map, which calls the top
scene's vtable `+0x150` = `0x004B6A90(nFlags, zDelta)`. That accumulates the delta in
`+0x160`, converts it to notches (divide by 120), then walks the scene's scrollbar list
`+0x144` (head `+0x14C`) and, for the first **visible** bar whose `+0x380` contains the
cursor (`+0xA0`/`+0xA4`, `PtInRect`), calls `0x00461320(bar, ±notches * step)`. Nothing
else needs to happen: the controller sees the moved position on its next poll.

## The list controller

Constructor `0x00430F90` (thiscall, no arguments), vtable `0x0066CFC8`, `0x3B4` bytes,
the bar embedded at `+0x8`. **`0x00430FE0` is its destructor** (slot 0 `0x00431330`
calls it).

|Field|Meaning|
|-|-|
|`+0x384`|visible rows (the bar's range)|
|`+0x398`|shown flag|
|`+0x39C`|item count|
|`+0x3A0`|the bar position last read|
|`+0x3A4` / `+0x3A8`|**first** / last visible row|
|`+0x3AC`|pending fine offset while a button is held|
|`+0x3B0`|last movement|

|Function|Signature|What|
|-|-|-|
|`0x004311B0`|thiscall(n), `ret 4`|set the item count: shows the bar when `n` exceeds the visible rows and hides it otherwise. **Only when the shown state or the count changed** does it push the total (`0x00461CA0(range * n)`), re-read the position and recompute the rows; a bar re-placed with the same count keeps a stale position. The ship overview hides the bar before every call (`0x004757FD`), which forces the resync|
|`0x00431260`|thiscall() -> bool|poll: read the bar; when moved, recompute the rows and return true. Otherwise, with a pending `+0x3AC` and no button held (`0x00461F80`), step the first row and write the position back (`0x00461DD0`)|
|`0x00431110`|thiscall()|recompute `+0x3A4`/`+0x3A8` from the position|

### Per-open sequence

What the ship overview does for each of its two bars on every open
(`0x004757E0`..`0x00475817`):

1. `0x00461480` (detach, in its build's first pass) and `0x00461FD0` (release the thumb);
2. `0x00460EF0(area, 0, step, 0)` - which registers bar and buttons after the window;
3. `+0xCC(0)` hide;
4. `0x004311B0(count)` - shows it if needed and resyncs.

Because the bar only submits its own rect when it moves, a parent that scrolls content
elsewhere must submit that content's rect itself (`0x004B9650`), or the rows repaint in
slivers.
