# Windows and Widgets

Everything drawn inside the game is a **widget**: a C++ object of the common base class
(constructor `0x004B15F0(name)`, vtable `0x006708D0`) or one of its subclasses -
buttons, scrollbars, panels, and the windows themselves. A **window** is a `CDialogBG`
(constructor `0x0041C320`, vtable `0x0066BC90`): a widget that paints the parchment. The
[UI chapter](../ui.md#window-class-family) lists the vtable slots; this page covers the
object layout, how a window gets drawn and receives input, and what a window of our own
needs. `p3-api`'s `ui::custom_window` implements the recipe.

## Widget layout

|Offset|Field|
|-|-|
|`+0x0`|vtable|
|`+0x14` / `+0x18`|x / y - screen coordinates for a child of the scene container|
|`+0x2C` / `+0x30`|width / height|
|`+0x41`|a byte set to 1 at construction and tested by the base event handler `0x004B1720` before it dispatches|
|`+0x48`|visible byte (1 after construction; `+0xC8` reads it, `+0xCC` sets it)|
|`+0x4C`..`+0x5C`|a Win32 `RECT`, empty after construction; the container copies it (`+0x84`) into the child entry's offsets|
|`+0x60`|an optional background widget drawn by the base draw `0x004B18F0` at offsets `+0x64`..`+0x6C`|
|`+0x70`..`+0x80`|four input handles, `-1` when unassigned|

`CDialogBG` adds the parchment fields: `+0x94` shadow flag, `+0x98`/`+0x9C` tile
counts, **`+0xA0` the tile map** (a heap array the base close frees). Derived windows put
their own members from about `+0xD0` (the ship overview's first is a list controller at
`+0xD0`).

Every constructor writes its class's vtable **before** running the shared initializer
`0x004B1650`, which dispatches through it (`+0x94(-1)`, `+0x104(-1, -1)`, `+0x114(0)`);
the same initializer is also called by the destructors. Since MSVC destructors write the
vtable first too, a vtable store at the top of a function is not evidence of a
constructor: check the class's vtable slot 0, the deleting destructor, which calls the
real destructor. In the scrollbar family, `0x00430FE0` and `0x00460A70` are the
destructors and die on a fresh buffer; the [button](./buttons.md)'s `0x004C6A30` is one
too (its constructor is `0x004C6910`), and the trading office window's constructor is
`0x005D8300`, its destructor `0x005D8590`.

## Common slots

The base class gives every widget the same handful of methods; a window positions and
shows its children through them, and so can a mod.

|Slot|Base|Meaning|
|-|-|-|
|`+0x38`|`0x00402460`|enabled: byte `+0x41`|
|`+0x40`|`0x00402470(out*)`, `ret 4`|size: copies `+0x2C`, `+0x30`, `+0x34` (width, height, depth); `+0x44` / `+0x48` return width / height through it|
|`+0x4C`|`0x004024B0(width, height, depth) -> 1`, `ret 0xC`|**set size**: `+0x2C`, `+0x30`, `+0x34`. The session UI init sizes every building window through it with hard-coded `425 x 510` (`0x00426D98`) and the [building backdrop](./building-backdrop.md) with `451 x 537`; nothing resizes them afterwards, so a mod can set a window's size before its open method lays it out|
|`+0x60`|`0x00402530(out*)`|position (a button adds its motion offset, `0x004C7D60`); `+0x6C` / `+0x70` / `+0x74` return x / y / z through it|
|`+0x64`|`0x00402550(x, y, z)`, `ret 0xC`|**set position**: `+0x14` / `+0x18`. The game's windows pass `0x64` for z|
|`+0x68`|`0x004C9270(point*)`|set position from a three-dword struct|
|`+0xC8` / `+0xCC`|`0x004B2200` / `0x004B21A0(bool) -> previous`|visible / show-hide; hiding also calls `+0xA8(0)`, `+0xB4(0)`, `+0xC0()`, and any change redraws through `+0xD4`|
|`+0xC4`|`0x004026E0`|focused: byte `+0x42` (see [Input](#input))|

## Drawing

The scene container calls a child's draw slot `+0x9C` with four stack arguments: the
**render context** it was itself given (an opaque value handed down the tree, `-1` at the
root), then the child entry's `x`, `y`, `z` offsets - zero for a direct child of the
scene, so `+0x14`/`+0x18` are screen coordinates. The ship overview's draw
(`0x00473220`) shows the contract:

1. rect = `(+0x14 + x, +0x18 + y, + width, + height)`; return unless `+0xC8` (visible);
2. `0x004BB7C0(&rect)` - return when it misses the dirty region (see
   [Submitting Screen Areas](../ui.md#submitting-screen-areas));
3. `0x004BB9B0(context)`, `0x004BB870(-1)` - renderer state, cdecl;
4. `0x0041CAA0(this, &rect)` - the parchment;
5. its own text and widgets.

**The parchment** is the shared `[DialogBG0]` resource (`TexID=16001`, shadows
`16051`/`16052`, `TileSize=32`; loaded once at startup by the sole caller `0x004272F4`
into `[0x006CBAF4]`). `0x0041CAA0` (thiscall(rect*), `ret 4`) tiles it into the rect in
whole 32-pixel tiles and draws nothing while the window's tile map `+0xA0` is 0; the map
comes from `0x0041C5A0(this, width, height, shadow_flag)` (`ret 0xC`), which also sets
`+0x94`, `+0x98`, `+0x9C`. A size that is not a multiple of 32 leaves an unpainted strip
at the right and bottom where the scene shows through. The ship overview calls it with
its own `+0x2C`, `+0x30` and 0 (`0x0047553D`) and rounds its x up to a multiple of 4
(`0x0047551C`).

The open-window list's per-frame slot `+0x124` posts the window's opaque backdrop:
`0x004BBAA0(x + s + f, y + s/2, x + w - s - f, y + h - s)` with `s = [0x006CBB08]` and
`f = [0x006CBAEC]` when `+0x94` is set (`0x004749C0`).

## Input

The container's input dispatcher calls the event slot `+0x18(point*, type)` (`ret 8`) on
the topmost child under the cursor, with the point in screen coordinates. The dispatcher
(call sites `0x004B5F3A`..`0x004B6A47`) sends four types: `1` is the **press**, sent to
the child the container records as pressed (`+0xC8`); `-2` is the **left release** (the
ship overview's row click, `0x00477150`; play-verified as the header click in
`mod-tavern-details`, and what a [button](./buttons.md) counts as its click); `2` and `-3`
are dispatched from its other paths and not yet pinned down. The widget base handler
`0x004B1720` is the default.

**Keyboard focus** is the container's too. Its click dispatcher (the function around
`0x004B5D80`) keeps the index of the last clicked child in `+0xC4`; when it changes, the
old child gets `+0xC0` (focus lost) and the new one `+0xBC` (focus gained). The text
widgets (constructor `0x004C9790`, vtable `0x00671890`) implement the pair as setting and
clearing byte `+0x42` (`0x004CA640` / `0x004CA6F0`, which also keep the global
`[0x006DD5DE]`), and `+0xC4` reads that byte back. Keys travel `WM_KEYDOWN` (MFC handler
`0x004C0BE0`) -> window stack `0x004B92E0(vk, repeat, flags)` -> the top scene's key slot
**`+0x1C`** -> the same slot on the `+0xC4` child, provided its `+0xC4()` is true
(`0x004B6BD8`). Every widget therefore has a key handler `+0x1C(vk, repeat, flags)`,
`ret 0xC`; the base's (`0x004B1850`) does nothing, the
[number box](../ui.md#number-widgets)'s filters digits.

The mouse wheel arrives through the MFC message map (`WM_MOUSEWHEEL` entry at
`0x00671118`, handler `0x004C0FA0`), goes to the top scene's slot `+0x150` = `0x004B6A90`,
and from there to the [scrollbar](./scrollbar.md) whose catchment rect contains the
cursor.

## Ini-defined windows

Windows load their layout from ini sections named after the class: the ship overview is
`CShipList`, section `ShipList%u` (`0x00472C60`, its `+0x8`), read through the ini manager
`0x004BE0B0(this = 0x006DD524, section, key, default, ini)`. The inis (`parchment.ini`,
`sidemenu.ini`, `buttons.ini`, ...) are not on disk: they sit as plain text inside
`p2arch0_eng.cpr`, under the `./scripts/` paths the code passes.

## A window of your own

The ship overview (singleton `[0x006CC950]`, constructor `0x004726B0`, vtable
`0x0066ECF8`, open `0x00475E40(params)`) is the model: nothing ties a window to a
building except who calls its open. The sequence that puts any window on screen, in a
town or on the world map:

1. construct: a buffer through `0x0041C320`, then your own vtable (a copy of `0x0066BC90`
   with `+0x8`, `+0x9C`, `+0xF4`, `+0x118`, `+0x124` replaced - the first and last are
   pure virtuals in the base), the rect, and `0x0041C5A0(w, h, 0)`;
2. open: `0x00462280(0)` to close the others; `0x004B4E30` on the top scene
   (`0x004B9730(0x006DA5F0)`) for the window, then for each sub-widget; `0x00462390`;
3. per frame the scene draws and updates it, the list calls `+0x124`, ESC and right-click
   reach `+0x118`;
4. close: `0x00462320`, the sub-widgets out, `0x004B4EB0` for the window.

`p3-api`'s `ui::custom_window::GameWindow` does exactly this behind a Rust trait, and
`mod-crash-reporter`'s F9 probe is the worked example.
