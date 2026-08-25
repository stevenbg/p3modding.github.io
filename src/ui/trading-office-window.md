# Trading Office Window
The trading office window object is held in the static `0x006E557C`; its vtable is at
`0x00679CB0` (module offset `0x279CB0`).

|Field|Meaning|
|-|-|
|`+0xECC4`|selected page, 0-6 (4 = the administrator "Trading Office" view)|
|`+0xECC8`|town index, copied from the town scene on open (`0x005D8E3A`)|

`select_new_page` (`0x005D9A20`, thiscall(this, page)) switches the side menu page and
rebuilds the direction arrows and price displays of the administrator view - but not
the amount displays.

## Administrator Amount Rows
The administrator view's per-ware rows are widget structs at
`window + 0x9840 + row * 0x190`, rows in the ware display order of the table at
`0x00698538` (identity in the executable, sorted at runtime by localized ware name).
Each row embeds a [number widget](../ui.md#number-widgets): the displayed amount lives
at `row + 0x188` and is written through the setter `0x0045C930`.

The amounts are populated only by the window's open method (`0x005D8950`,
vtable `+0x120`): its 20-ware loop at `0x005D8F40` reads the administrator's minimum store
quantity (`office + 0x354`, the field [operation `0x5B`](../operations/005b-office-autotrade-setting-change.md) writes), divides by the ware scaling (barrels 200, bundles 2000, via the
scaling table at `0x00672C14`), clamps to 9999 and calls the widget setter. This is
why administrator amounts historically refreshed only when the window was reopened;
a mod can refresh them in place by re-running the same computation against the row
widgets.

Re-running the open method itself repopulates everything but registers the window
family with the [window manager](../ui.md#window-manager) a second time; pairing it
with the close method (`0x005D92C0`, vtable `+0x118`) balances the registration but
detaches the side menu - the game's real open path goes through a view controller
above the window.

## Lock Checkbox
The per-ware "Lock min. store quantity for auto trade ships" checkbox is drawn
directly from the office lock bitmap (`office + 0x3B4`): the draw code at
`0x005D9D98`/`0x005DD987` resolves the office through the lookup at `0x005308A0`,
passing the player merchant global (`operations + 0x924` = `0x006DFC14`) and the
window's town - establishing that lookup's argument order as (merchant, town). Toggling
the checkbox goes through
[operation 0x66](../operations/0066-office-autotrade-lock-change.md).
