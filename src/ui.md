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
|`0x006E5574`|tavern window|
|`0x006E557C`|trading office window|
|`0x006E558C`|town hall window|
|`0x006E55C0`|shipyard window|
|`0x006CBA74`|auto trade goods dialog ("Automatic maritime trading")|
|`0x006E51AC`|local map scene (town view AND sea battle - see below)|

About twenty more statics in the `0x006E5500`-`0x006E55D0` cluster hold further
windows. The store is not part of the constructor: the constructor takes `this` in ecx
and returns it in eax, and the mass-constructor's call site stores that into the
static - for the tavern window, constructor `0x005CB9B0` called at `0x00426C2C`,
`mov ds:0x006E5574, eax` at `0x00426C45`. A shutdown path around `0x00427F00` destructs
the objects and writes zero back into the statics (`0x00427F74` for the tavern).

A window's own methods never read its static; the code that opens the window does. So
searching a window's address range for one of these statics finds nothing, and the way
to identify a static is to disassemble the mass-constructor around the call to the
window's constructor.

Not every UI object has a static: the scrollmap's trade route panel, for example, is
only reachable through its vtable (see [Trade Route Panel](./ui/trade-route-panel.md)).

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
A singleton at `0x006DA5F0` (also reachable through `0x004B9730`) tracks the open
windows in two containers.

The registration list (`this+0xC0`): `0x004B4E30` registers a window, `0x004B4EB0`
deregisters it. Windows register their embedded sub-windows too. Calling a window's
open method on an already-open window registers it twice - it then draws twice and
needs two closes - so programmatic refreshes must not re-run open (see
[Trading Office Window](./ui/trading-office-window.md) for the working alternative).

The **window stack** (an MFC-style list at `this+0x4`): `+0xC` points at the TOP
node, `+0x10` holds the depth, and each node is `{+0x4: link toward the bottom,
+0x8: the window object}`. Windows enter through the push method
`0x004B90E0(window, arg)` (activates via vtable `+0xD4`/`+0x15C`, inserts the node
via `0x0064E6FC`) and leave through the remove method `0x004B9150(window)` (finds
the node from the top, unlinks it via `0x0064E749`, notifies via vtable
`+0xD4`/`+0x160`) - both thiscall on `0x006DA5F0`, with 47 and 43 call sites.

The stack is what runs the game: the main loop is
`while (0x004B8A40(this = 0x006DA5F0) != -1)` (the loop itself at `0x004B70C0`),
and each frame that method pumps messages, updates the
[frame clock](./basics/time.md#the-frame-clock) and calls the TOP window's vtable
`+0xF4` (update) and `+0x12C` (`0x004B8B0D`). Scenes - scrollmap, town view, sea
battle - are window objects on the same stack as the building windows and dialogs,
so "which scene is the player looking at" is a read of the top node.

## The Local Map Scene
One window object serves both the town view and the sea battle - what differs is
the map loaded into it. It is allocated at `0x00424DD4` (0xCBA8 bytes, constructor
`0x00586FF0`), kept in the static `0x006E51AC`, and carries two vtables: the main
one at `0x00677998` and a second interface at object `+0x94` (`0x00677990`). The
main vtable ends around `+0xF8` - unlike the building windows there are no
close/open slots at `+0x118`/`+0x120`.

Its per-frame update (`+0xF4` = `0x0058B7F0`) drives the entire scene frame -
simulation, battle AI, the wind (`0x006113C9`), the changed-rect submit
(`0x004B9650`) - and paces the simulation purely by the
[frame clock](./basics/time.md#the-frame-clock): neither the game tick nor the
call count matters (calling the update several times per frame moves nothing).

`+0xC324` holds the loaded map's id. Two loaders write it - `0x0058A733` stores the
id as given, `0x0058A395` sets bit `0x80` first (`or al,0x80`) - and `0xFF`/`-1`
mean no map (`0x00589DEE`, `0x0058B590`); the scene's own update starts with
`and eax,0x7F` and a compare against the town count to pick the town record. The id
space is only partly mapped, and "is a battle running" is NOT decidable from it:
towns attacked from the sea fight on the town's own map. The map files
(`iso/towns/<id>.*`) come as ids 0..30 (the towns), 128..155, 201..205 (five -
matching `SeaBattleShaderCnt=5` in `scripts/iso.ini`) and 251..255; which class
means what has not been pinned down.

## Window Titles
`0x00420C70` (stdcall, arguments: a string object and the window) draws a window's
title banner. It fetches graphic `0x791E` from the resource manager at `0x006DA820`
(`0x004B3DD0`), takes the four dwords of its rect and renders it together with the
text. Pages that show no title simply never call it, which leaves the strip at the
top of the window free - `mod-tavern-details` uses it for table rows.

## Submitting Screen Areas
`0x004B9650` takes one argument by stdcall: a pointer to four dwords - left, top,
right, bottom. It returns without doing anything while `[0x006DCB94]` is non-zero,
or when the rect's width or height is zero. Otherwise it iterates the pointer array
at `0x006DCD20` (`[0x00670F6C]` entries), passing each entry to `0x004BB780` and the
rect to `0x004BB140` - both trampolines into `ddraw_Dll`. The function takes no
object of its own and is called from 585 places in the executable.

The town hall window calls it for its own rect from its update method (vtable
`+0xF4`, `0x005E0850`), when the timestamp at `window + 0x1930` is older than the
date serial `0x006DE4B4` (compared at `0x005E08A6`); `mod-town-hall-details` zeroes
that timestamp so the call happens while its page is open. The trading office
window's update method (`0x005D9500`) contains no such call.

Observed while adding a text page to the trading office window (see
`mod-trading-office-details`): with no such call for the window's rect, the area
shows a mix of old and new pixels until something else submits it - moving the mouse
across it, or alt-tabbing out and back. Making the call from inside the window's draw
method (`+0x9C`), once or on every frame, leaves the background art torn and the text
flickering. Making it from the window's update method renders the page cleanly.

## Rich Text
Prose - letter bodies, the tavern's side room, anything that needs word wrap or inline
symbols - is drawn by a text-layout class of its own (vtable `0x0066E36C`, constructor
`0x004624D0`). Windows that need it own an instance: the tavern keeps one at
`window + 0x1608`, the town hall at `window + 0x18C8` (used at `0x005E40C2`, in a
function that also writes the window's `+0x1930` refresh timestamp).

|Function|Signature|
|-|-|
|`0x00420A10`|thiscall(layout, string, x, y, width, height, color) - draws|
|`0x00462520`|thiscall(layout, string, width, 0) - lays out, called by the above|

The layout pass fills a vector of 28-byte line records at `layout + 0x10`, with the count
at `+0x14`; the draw pass walks them. The string argument is a
[string object](#string-objects) passed by value as one dword: construct it with
`0x0064F2C1` and do **not** destroy it, because the draw destroys the parameter itself
(it calls `0x0064F253`).

`width` is only the wrap limit. What `x` anchors depends on the line's alignment, which
the draw turns into an offset at `0x00420A97`-`0x00420AB7`: nothing for a left-aligned
line, `(width - line) / 2` for a centred one, and **minus the line's own width** for a
right-aligned one. So `x` is where the text starts under `\l` and where it ends under
`\r`.

### Markup
The layout pass recognises exactly these escapes; anything else after a backslash is
literal text.

|Escape|Meaning|Handled at|
|-|-|-|
|`\l` `\r` `\c`|align the line left, right or centre|`0x00462659`, `0x00462674`, `0x0046268B`|
|`\f`|select a font - letters open with `\f1_`|`0x004626A2`|
|`\t`|tab, taking an `_`-delimited argument|`0x00462764`|
|`\h`|substitution, `_`-delimited|`0x00462774`|
|`\C`|coin symbol, from `[0x006CC37C]`|`0x004627F5`|
|`\L`|cargo (load) symbol, from `[0x006CC384]`|`0x00462804`|
|`\B`|barrel symbol, from `[0x006CC380]`|`0x00462813`|
|`\d` + `A`..`Z`|the decorated initial capital for that letter, from the table in `[0x006CC3D4]`|`0x004626C5`|

The symbols are graphics, not font glyphs: the escape takes the handle from `+0x4` of the
object in that global and measures it with `0x004BBB20` - the same call the icon blits use
(see [Graphics and Icons](#graphics-and-icons)) - so the layout can flow the text around
it. The `\d` family is the drop caps a
document starts with - all 26 letters exist.

### The `\d` Off-By-One
`\dX` is three characters, but its branch advances the input pointer by two
(`add ebp,2` at `0x00462758`), where every other escape advances by its own length
(`inc ebp` for `\l` at `0x00462667`, `add ebp,2` for the two-character `\C` at
`0x004627A3`). The literal segment that follows therefore starts one byte past its end,
and the copy that flushes it computes its length as `end - start` without guarding
against a negative result:

```
00462CD5  sub  ebx, eax        ; length = end - start  ->  0xFFFFFFFF
00462CD7  je   done            ; only exactly zero is handled
00462CD9  lea  ecx, [ebx+1]    ; malloc(0)
00462CE7  test ebx, ebx
00462CE9  jbe  done            ; unsigned, so -1 is not <= 0
00462CEB  ...                  ; copies 4GB into a 0-byte buffer
```

A `\dX` at the very end of a string therefore crashes the process with an access
violation in that copy. With any text after the escape the length stays positive and it
renders, at the cost of one following character being swallowed - so a `\dX` wants a
spare character behind it.

## Graphics and Icons
The small icons the building pages put beside their numbers - a coin, a crate, the crew
figure - are graphics fetched by id from the resource manager at `0x006DA820` and blitted.
The sequence, as the tavern does it at `0x005CDD07`, the shipyard at `0x005F4ECE` and
[render_window_title](#window-titles) at `0x00420C8C`:

|Step|Call|Notes|
|-|-|-|
|fetch|`0x004B3DD0` thiscall(manager, id)|returns the graphic record, or 0|
|check|record `+0x14` > 0 and `+0x4` != 0|the guards the game itself applies; `+0x14` is the frame count and `+0x4` the handle the renderer takes|
|measure|`0x004BBB20(handle, &size)`|writes width then height as two dwords - of the whole texture, see below|
|select|`0x004BB9C0(handle)`|the way `0x004BB8F0` selects a font|
|colour|`0x004BB870(0xFFFFFFFF)`|**required** - see below|
|blit|`0x004BB330(src_x, src_y, x, y, width, height)`|cdecl, six arguments|

The blit **modulates the image by the constant colour**. Drawing an icon while a text
colour is still set produces a silhouette in that colour rather than the picture, so the
colour has to be set to `0xFFFFFFFF` first - and set back afterwards by whatever draws text
next.

### Where the Ids Come From
The ids are not constants in the executable. Each window reads them by name through the
ini lookup `0x004BE0B0(section, key, file, ...)` on the store at `0x006DD524`, and caches
them in its own fields - the shipyard keeps its at `window + 0xC94` onward. The names live
in `scripts/parchment.ini` and `scripts/BuildingParchment.ini` inside `p2arch0_eng.cpr`,
one section per building: `[Werftparchment]` for the shipyard, `[Kneipeparchment]` for the
tavern, `[Kontorparchment]` for the trading office, and so on.

Vanilla 1.1 values, from `[Werftparchment]`:

|Key|Id|Icon|
|-|-|-|
|`WarenID`|16046|wares|
|`KohleID`|16047|money|
|`KonvoiID`|16043|convoy|
|`KapitaenID`|16053|captain|
|`BewaffnungKleinID`|16054|small armament|
|`BewaffnungGrossID`|16056|large armament|
|`HerzID`|20013|heart|
|`KnotenID`|20016|knots|
|`CrewID`|20017|crew|
|`TimeID`|32001|hourglass|

`scripts/textures.ini` then maps an id to its picture: `[TEX20017]` is
`images/frames_listen/crew0002.tga` with `OffsetNSize0 = 0 0 26 18` - which is where the
blit's source offset and size arguments come from.

### Sheets and Frames
One id can hold several pictures. The record's `+0x14` is the frame count and its `+0xC`
points at an array of four-dword rects - source x, source y, width, height - one per frame,
which are exactly the `OffsetNSize0`, `OffsetNSize1`, ... entries of the ini. Blitting a
frame means blitting its rect out of the shared texture, so the side menu's three
skill-bonus icons come from `[TEX20011]`, `images/sidemenu/bonus.tga`, `Count=3`, as three
16x16 frames at (0,0), (16,0) and (0,16).

This is why measuring is the wrong way to size an icon: `0x004BBB20` reports the whole
texture, which for a sheet is every frame at once. The frame's own rect is the size to use,
and the icons are not one size anyway - 16x16 bonus frames, an 18x18 captain, a 26x18 coin,
crew figure and pirate - so anything placing text against an icon has to read its rect.

Three of these icons are also reachable as [markup](#markup) escapes - `\C`, `\L` and
`\B` - which is the better route when the icon belongs *inside* a line of text, since the
layout measures it and flows the text around it. A blit is absolutely positioned.

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
