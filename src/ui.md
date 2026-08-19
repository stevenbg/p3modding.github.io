# UI
P3's user interface is built from window objects sharing a common class family, managed
by a central window manager. This chapter collects what has been reverse engineered
about the framework and individual windows.

## Window Objects
Most windows are constructed once at startup by a mass-constructor around `0x00426000`
and live for the whole session; "opening" and "closing" only registers and deregisters
them with the window manager. Many hold their object pointer in a static:

|Static|Window|
|-|-|
|`0x006E5500`|town hall side menu|
|`0x006E557C`|trading office window|
|`0x006E558C`|town hall window|
|`0x006E55C0`|shipyard window|
|`0x006CBA74`|auto trade goods dialog ("Automatic maritime trading")|

About twenty more statics in the `0x006E5500`-`0x006E55D0` cluster hold further
windows, each written exactly once by its constructor. Not every UI object has a
static: the scrollmap's trade route panel, for example, is only reachable through its
vtable (see [Trade Route Panel](./ui/trade-route-panel.md)).

## Window Class Family
The window classes share their vtable layout. Two slots are load-bearing for modding:

|Vtable slot|Method|
|-|-|
|`+0x118`|close: deregister from the window manager, hide|
|`+0x120`|open: register with the window manager, build/populate the widgets|

Verified for the trading office window, the town hall window and the goods dialog
(base class vtable `0x0066BC90`, base open `0x00462390`). Hooking these slots is the
established way to track a window's open state (used by
`mod-trading-office-prices-synchronization` and `mod-auto-supply`).

## Window Manager
A singleton reachable through `0x004B9730` (`this = 0x006DA5F0`) tracks the open
windows: `0x004B4E30` registers a window, `0x004B4EB0` deregisters it. Windows
register their embedded sub-windows too. Calling a window's open method on an
already-open window registers it twice - it then draws twice and needs two closes -
so programmatic refreshes must not re-run open (see
[Trading Office Window](./ui/trading-office-window.md) for the working alternative).

## Number Widgets
The numeric row widgets (amounts, prices) cache their displayed value and text. The
setter at `0x0045C930` (thiscall, one argument) clamps the value to the widget's
bounds at `+0x180`/`+0x184`, stores it at `+0x188`, flags `+0x18C` dirty and rewrites
the label text. In-place writes to the underlying data are invisible until either this
setter runs or the owning window repopulates.

## String Objects
Several game functions take an MFC-style string object instead of a plain C string: a
single pointer to character data whose header (refcount, allocated size, length) sits
in the 12 bytes before the data. Passing a raw `char*` to such a function crashes -
the callee dereferences the characters as a pointer.

- construct/assign from a C string: `0x0064F390` (thiscall(this, char*))
- destruct: `0x0064F253` (thiscall(this))
- `[0x006C7CCC]`/`[0x006C7CD0]` hold the shared empty-string sentinel; a fresh object
  should be initialized to `[0x006C7CD0] + 0xC` so the constructor's release-old-data
  path is a no-op.

The trade route file loader (`0x004D5EE0`, see
[Trade Routes (.rou)](./file-formats/rou.md)) is one such consumer.

## Render Imports
Drawing goes through the game's own render DLL, `ddraw_Dll.dll` (shipped in the
game directory, distinct from the system's `ddraw.dll`). Its exports are bound at
startup into a function-pointer table in BSS around `0x006DA9F0`-`0x006DAA10`,
and the game calls them through a block of trampolines at `0x004BB3E0` onward,
one `jmp [pointer]` each - e.g. `0x004BB3F0` is `jmp [0x006DAA04]`, the text
draw.

That text draw (`ddraw_Dll+0xF100`, cdecl, fourth argument the C string) begins
with `cmp byte [string], 0` - no validity check of any kind - so any bad string
pointer the game passes crashes *inside* `ddraw_Dll`. A crash address in
`ddraw_Dll` therefore usually means bad arguments from game code, not a render
bug; the caller is on the stack right above (see the
[patrol letter crash](./bugs/patrol-letter-crash.md) for a worked example).
