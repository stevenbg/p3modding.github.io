# Trading Office Window
The trading office window object is held in the static `0x006E557C`; its vtable is at
`0x00679CB0` (module offset `0x279CB0`). Constructor `0x005D8300`, destructor
`0x005D8590`, open `0x005D8950` (vtable `+0x120`), close `0x005D92C0` (`+0x118`), draw
`0x005D95A0` (`+0x9C`), update `0x005D9500` (`+0xF4`), ini load `0x005D9710` (`+0x8`,
section `[Kontorparchment]` of `./scripts/BuildingParchment.ini`).

|Field|Meaning|
|-|-|
|`+0xECC4`|selected page, 0-6 (4 = the administrator "Trading Office" view); `-1` is the empty page the window opens on|
|`+0xECC8`|town index, copied from the town scene on open (`0x005D8E3A`)|

`select_new_page` (`0x005D9A20`, thiscall(this, page)) switches the side menu page and
rebuilds the direction arrows and price displays of the administrator view - but not
the amount displays.

## Layout

The window is built from the stock [widget classes](./windows-and-widgets.md); the
constructor's `__ehvec_ctor` calls lay out the arrays, the open method clones every
[button](./buttons.md) from its template, positions everything and adds it all to the
top scene (window first, then the fixed buttons, then row by row), and the close method
removes it all again.

The window is **425 x 510**, set once at startup through the size slot (`+0x4C`,
`0x00426D98`) like every building window, over the 451 x 537
[backdrop](./building-backdrop.md). The open method centres the window on the town view
from that size (`0x005D89A6`..`0x005D89CC`) and lays each row out **from the right edge
inwards**: starting at `x + width - 25` and stepping left by the widgets' own widths plus
fixed gaps (`0x005D8E9F`..`0x005D8EF3`), so every widget in the row - the price `+`, box
and `-`, the lock button, the amount `+`, box and `-`, the arrows - gets its x from the
width, while the ware names are text drawn by the page at a fixed offset from the left
edge. A larger width written into `+0x2C` before the open method runs therefore moves the
row widgets outwards and opens the extra room on the left of them; `mod-trading-qol` does
that, together with enlarging the backdrop behind it.

|Offset|Count x size|Class|Role|
|-|-|-|-|
|`+0x94`|1|`0x0040A440`|the chart image (`ChartFile`)|
|`+0xD0`|1 x `0xE8`|button, template 5|**the X**: at `(x + w - 32 - 3, y + h - 18 - 3)`; the update polls it at `0x005D955D` and closes the window|
|`+0x1B8` `+0x2A0` `+0x388`|3 x `0xE8`|button, template 4|huge caption buttons|
|`+0x470` / `+0x558`|2 x `0xE8`|button, template 0|`-` / `+` under the table, type `permanent`|
|`+0x73C` / `+0x195C`|20 x `0xE8` each|button, template 0|per row `-` / `+`, type `permanent`|
|`+0x2B7C` `+0x3D9C` `+0x4FBC` `+0x61DC` `+0x73FC`|20 x `0xE8` each|buttons|per row: the direction arrows (`+0x61DC` shown = buy, `+0x4FBC` shown = sell), min/max|
|`+0x861C`|20 x `0xE8`|button, template 0|**the lock checkbox's button**|
|`+0x9840` / `+0xBA00`|20 x `0x190` each|[number box](../ui.md#number-widgets)|amount / price|
|`+0xB780` / `+0xD940`|20 x `0x20` each|`0x004B03E0`|per-row auxiliaries|
|`+0xDBC0`|20 x `0xD8`|[image](./image-widget.md), `[ANIM12]` of `BuildingParchment.ini`|**the lock checkmark**|

The rows follow the ware display order of the table at `0x00698538` (identity in the
executable, sorted at runtime by localized ware name); row `n` sits at
`y = window.y + 0x46 + n * 0x14` (`0x005D8E2B`, `0x005D9237`), every widget placed with its
top-left on that line through slot `+0x64(x, y, 0x64)`.

## Administrator Amount Rows
Each amount row embeds a [number widget](../ui.md#number-widgets): the displayed amount lives
at `row + 0x188` and is written through the setter `0x0045C930`.

The amounts are populated only by the window's open method: its 20-ware loop at
`0x005D8F40` reads the administrator's minimum store quantity (`office + 0x354`, the
field [operation `0x5B`](../operations/005b-office-autotrade-setting-change.md) writes),
divides by the ware scaling (barrels 200, bundles 2000, via the scaling table at
`0x00672C14`), clamps to 9999 and calls the widget setter. This is why administrator
amounts historically refreshed only when the window was reopened; a mod can refresh them
in place by re-running the same computation against the row widgets.

Re-running the open method itself repopulates everything but registers the window
family with the [window manager](../ui.md#window-manager) a second time; pairing it
with the close method balances the registration but detaches the side menu - the game's
real open path goes through a view controller above the window.

## Typing and the Per-Frame Commit
The page-4 update (`0x005DCDB0`, reached from the update's page jump table) polls every
row's buttons with `0x004C78B0` and, at `0x005DD0CB`, asks the row's amount box and price
box whether they are focused (`+0xC4`). For a focused row it enqueues operation `0x5B`
**every frame**: the amount box's `+0x188` scaled by `0x00672C14`, and the price box's
`+0x188` signed by the arrow buttons (`0x005DD4B6`: the `+0x61DC` arrow visible makes it a
buy, negative; `+0x4FBC` a sell, positive; neither, 0). The boxes never commit
themselves, so anything that changes a focused box's value has changed the order.

## Lock Checkbox
The per-ware "Lock min. store quantity for auto trade ships" checkbox is two widgets: the
round button in `+0x861C` and the checkmark image in `+0xDBC0`, registered right after it
and placed at `(button.x - 4, button.y)` (`0x005D9038`..`0x005D9053`). Entering the page
sets each checkmark's visibility from the office lock bitmap (`office + 0x3B4`, read at
`0x005D9D98`): the code resolves the office through the lookup at `0x005308A0`, passing
the player merchant global (`operations + 0x924` = `0x006DFC14`) and the window's town -
establishing that lookup's argument order as (merchant, town). A click on the button
(`0x005DD13A` -> `0x005DD434`) inverts the checkmark's visibility and enqueues
[operation 0x66](../operations/0066-office-autotrade-lock-change.md) with the new state.
