# Buttons

Every clickable control in the game - the X that closes a window, the arrows and `+`/`-`
of a ware row, the round base of a checkbox - is one class, `CViperButton` (name at
`[0x006BF85C]`): four looks, a caption and one of three behaviours. `p3-api`'s
`ui::button` wraps it; `mod-trading-qol` puts twenty-one of them on the trading office
window.

## The class

Constructor `0x004C6910` (thiscall, no arguments - it reads its class name itself),
vtable `0x006714C8`, `0xE8` bytes. **`0x004C6A30` is the destructor**: vtable slot 0
(`0x00408B70`) hands it to `__ehvec_dtor`, and it destroys the two strings and runs the
base destructor chain. Like every destructor in this widget family it writes the vtable
first, so it reads like a constructor; called on a fresh buffer it corrupts the `CString`
allocator (the crash surfaces at the next string allocation, `0x0064F0E7`).

|Field|Meaning|
|-|-|
|`+0x4` / `+0x8`|current / previous state, 0..3, set by vtable `+0x2C` = `0x004C7C50(state)`|
|`+0x94` `+0x98` `+0x9C` `+0xA0`|the four looks, heap [image widgets](./image-widget.md), indexed by state|
|`+0xA4`|motion object (`0x24` bytes, constructor `0x004ADF00`)|
|`+0xA8` / `+0xAC`|`CString` caption / second caption|
|`+0xB8`|**type**: 0 `normal`, 1 `check`, 2 `permanent` (the loader matches the ini's `Type` against the names at `0x006BF750` and stores the values at `0x006714B8`)|
|`+0xBC`|**clicked** flag|
|`+0xBD`|pressed-inside flag|
|`+0xD8`..`+0xE4`|autorepeat timing of a `permanent` button (frame counter `[0x006DCCF8]`)|

|Function|Signature|What|
|-|-|-|
|vtable `+0x8` = `0x004C6D20`|thiscall(CString by value, id), `ret 8`|load `[Button<id>]`: the four looks from `Disable`/`Neutral`/`Pressed`/`Hover` (ids in `DFile`/`NFile`/`PFile`/`HFile`), `Motion`/`MFile`, `Type`, `Size`. The string is destroyed by the callee|
|`0x004C7A90`|thiscall(this = source, dest), `ret 4`|**clone**: the widget base copy `0x004B1D20` (position, size, flags), fresh copies of the four looks (`0x004ADA30`) and the motion (`0x004AEA50`), both captions, type and flags|
|`0x004C7780`|thiscall(char*), `ret 4`|set the caption and redraw|
|`0x004C7C30`|thiscall(type), `ret 4`|set the type (values above 2 are ignored)|
|`0x004C78B0`|thiscall() -> bool|**clicked**: return `+0xBC` and clear it; a `permanent` button also reports while held, at its delay|
|vtable `+0x120` = `0x004C7940`|thiscall() -> bool|**checked**: the current look's `+0x10` flag|
|vtable `+0x124` = `0x004C7970`|thiscall(bool), `ret 4`|set checked, redrawing (`+0xD4`) on change|
|vtable `+0x18` = `0x004C6B90`|thiscall(point*, type), `ret 8`|the event handler, below|

Nothing polls a button for you: its owner calls `0x004C78B0` each frame. The trading
office's update asks its X button at `0x005D955D` and closes itself on true.

## The event handler

`0x004C6B90` first runs the widget base's `0x004B1720` (the hover/press state machine
behind `+0x4`), then, if the button is enabled (`+0x38`):

- event `1` (the press the container sends to its pressed child): `+0xBD = 1`;
- **`normal`** and **`check`**: on `-2` (left release) inside the button (vtable slot `+0xAC` =
  `0x004B23A0`, which reads byte `+0x43`) with `+0xBD` set, `+0xBC = 1`; a `check` button also toggles itself
  (`+0x124(!+0x120())`). `+0xBD` clears on `-2` or `0`;
- **`permanent`**: `+0xBC = 1` on the press (resetting the autorepeat timers), cleared on
  the release; the look's pressed flag follows the same two events.

## The template table

Windows never load a button from an ini. At startup `0x00424DA4` loads `[Button0]` to
`[Button5]` of `./scripts/buttons.ini` (`0x0066DE5C`, count byte `[0x0066DE5A]` = 6) into
a table at **`0x006CBE08`**, stride `0xE8`, and every button on screen is a clone of one
of them through `0x004C7A90`, re-captioned and moved into place. The ids the trading
office uses sit in the byte table `0x0066DE54..`: `[+0]` 4, `[+1]` 3, `[+2]` 2, `[+3]` 1,
`[+4]` 0, `[+5]` 5.

|id|Section|Size|Look|
|-|-|-|-|
|0|`[Button0]`|16 x 18|round - the checkbox base, the row arrows and `+`/`-`|
|1|`[Button1]`|32 x 18|small caption button|
|2|`[Button2]`|48 x 18|medium|
|3|`[Button3]`|96 x 18|large|
|4|`[Button4]`|128 x 18|huge|
|5|`[Button5]`|32 x 18|the X that closes a window (TexID 16023, frames 0/1/2)|

Since the table exists only after startup, a mod builds its buttons lazily - on the first
open of the window they ride on - not in `start()`.

## Putting one on a window

Clone a template, set caption and type, position it with the common slot `+0x64(x, y,
0x64)` (see [Windows and Widgets](./windows-and-widgets.md#common-slots)), and add it to
the top scene **after** the window so it draws above it and gets the clicks; remove it when
the window closes. The window's own open method does exactly this for each of its buttons
(`0x005D8950`: clone at `0x005D89FF`, place, `0x004B4E30`), and its close removes them
(`0x005D92C0`). A checkbox is a round button with an [image widget](./image-widget.md)
registered right after it.
