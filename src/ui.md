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

Not every static lives in that cluster, so failing to find one there does not mean
there is none: the scrollmap's
[ship panel](./ui/ship-panel.md) keeps its instance in `0x006CE6D0`,
stored by the same mass-constructor at `0x00426700`.

### Lifecycle Across Loads
All verified live with hooks on the open/close/destructor slots:

- **An in-game load (settings screen -> load game) keeps every UI object alive.**
  No destructor runs and the same window pointers serve the loaded game - state a
  window object carries silently survives into the loaded world. Only the menu
  path recreates the UI: quitting to the main menu runs the teardown around
  `0x00427F00`, and loading from the menu runs the mass-constructor again.
- **The teardown destroys windows without closing them** (destructor, flags 3, no
  `+0x118` call). That is harmless only because a building window cannot be open
  when the teardown runs: the settings screen - the only route to quit or load -
  closes building windows and dialogs before it opens (the close is logged before
  the settings screen is even pushed, on every path: ESC, the X button,
  right-click, clicking another building, the leave-town button). The goods
  dialog additionally receives defensive close calls during the teardown itself.
- A window's constructor does not necessarily initialize its fields: the church
  window's mode field (`+0x1D30`) has no writer in its constructor, and since the
  recreated window usually mallocs into the block the old one vacated, the field
  inherits the previous session's value - the mechanism behind the
  [church animation crash](./bugs/church-window-animation-crash.md).

## Window Class Family
The window classes share their vtable layout: every window derives from the parchment
widget `CDialogBG` (constructor `0x0041C320`, vtable `0x0066BC90`, 74 slots), itself a
widget of the common base (`0x004B15F0`, vtable `0x006708D0`). The slots that matter for
modding:

|Vtable slot|Method|
|-|-|
|`+0x8`|load the window's ini section - pure virtual in the base|
|`+0x18`|event handler `(point*, type)`, `ret 8` - see [Windows and Widgets](./ui/windows-and-widgets.md#input)|
|`+0x9C`|draw `(context, x, y, z)`, `ret 0x10` - called by the scene container each frame|
|`+0xC8`|is visible (base: byte `+0x48`)|
|`+0xCC`|show/hide `(bool)`|
|`+0xF4`|per-frame update - called by the scene container while visible|
|`+0x118`|close: leave the [open-window list](#the-open-window-list), free the tile map|
|`+0x11C`|may I close? `-> bool` (base `0x00602980`: always yes)|
|`+0x120`|open: join the open-window list (base `0x00462390`), build/populate the widgets|
|`+0x124`|per-frame call from the open-window list driver - pure virtual in the base|

Verified for the trading office window, the town hall window, the goods dialog, the
tavern window and the ship overview. Hooking `+0x118`/`+0x120` is the established way
to track a window's open state (`mod-trading-office-prices-synchronization`,
`mod-trading-qol`); `mod-tavern-details` hooks `+0x18` for header clicks. The slot table
for a class starts at its vtable, so a slot of one window is patched without touching
the others. [Windows and Widgets](./ui/windows-and-widgets.md) has the field layout, the
drawing contract and how to build a window of your own.

## Window Manager
A singleton at `0x006DA5F0` holds the **window stack**; the scene on top of it is a
widget container whose children are the building windows, dialogs and every one of
their sub-widgets.

`0x004B9730(this = 0x006DA5F0)` returns the top scene's object (`[[+0xC]+0x8]`, or 0
with an empty stack). That object is the container: its children sit in an array at
`+0x94` (16-byte entries `{object, x, y, z}`, count `+0xC0`), `0x004B4E30(child)`
appends one - taking the offsets from the child's `+0x84` rect - and `0x004B4EB0(child)`
removes one (a no-op when absent). Each frame the container calls every visible child's
update `+0xF4` (loop at `0x004B6E80`) and draw `+0x9C` (loop at `0x004B597D`) **in
array order**, and its input dispatcher gives a click to the topmost child under the
cursor, so a widget registered after a window draws above it and receives the clicks
inside it. Windows register their sub-widgets right after themselves; the ship
overview's build (`0x004751D0`) adds itself, then its buttons and both list scrollbars.
Calling a window's open method on an already-open window registers it twice - it then
draws twice and needs two closes - so programmatic refreshes must not re-run open (see
[Trading Office Window](./ui/trading-office-window.md) for the working alternative).

The stack itself (an MFC-style list at `this+0x4`): `+0xC` points at the TOP
node, `+0x10` holds the depth, and each node is `{+0x4: link toward the bottom,
+0x8: the window object}`. Windows enter through the push method
`0x004B90E0(window, arg)` (activates via vtable `+0xD4`/`+0x15C`, inserts the node
via `0x0064E6FC`) and leave through the remove method `0x004B9150(window)` (finds
the node from the top, unlinks it via `0x0064E749`, notifies via vtable
`+0xD4`/`+0x160`) - both thiscall on `0x006DA5F0`, with 47 and 43 call sites.

The stack is what runs the game: the main loop is
`while (0x004B8A40(this = 0x006DA5F0) != -1)` (the loop itself at `0x004B70C0`),
and each frame that method pumps messages, updates the
[frame clock](./time.md#the-frame-clock) and calls the TOP window's vtable
`+0xF4` (update) and `+0x12C` (`0x004B8B0D`).

**The stack holds scenes and full-screen menu screens only** - the scrollmap, the
town view, the main menu, the settings screen and its dialogs. Building windows
and dialogs never enter it: they are children of the scene, opened and closed
through their vtable `+0x120`/`+0x118` without the manager seeing them. Verified
live with entry detours on both transition methods: across thirteen
open/close cycles of the trading office window the stack depth never moved, and
push/remove fired only on scene swaps (entering and leaving a town exchanges the
town-view and scrollmap windows) and menu screens. "Which scene is the player
looking at" is therefore a read of the top node. The game also calls the remove
method defensively on windows that are not on the stack - the session init runs a
whole batch of such no-op removes - so a remove is not proof the window was ever
pushed.

## The Open-Window List
Which building windows and dialogs are open is tracked separately, in an MFC
`CPtrList` at `0x006CC3B4` (head node `0x006CC3B8`, tail node `0x006CC3BC` = the
topmost, count `0x006CC3C0`; a node is `{+0x0 next, +0x8 window}`). The base open
`0x00462390` appends the window and re-posts the mouse position over its rect; the base
close `0x0041CCF0` calls `0x00462320`, which submits the window's rect and removes it.

|Function|Signature|What|
|-|-|-|
|`0x004621F0`|cdecl()|per-frame driver: for `i = count-1 .. 0`, call that window's vtable `+0x124`|
|`0x00462230`|cdecl() -> bool|close the topmost window: `+0x11C` veto, then `+0x118`; on veto plays the refusal sound `0x1F41`|
|`0x00462280`|cdecl(bool force) -> bool|close every open window, same veto unless forced|
|`0x00462310`|cdecl() -> bool|is any window open|
|`0x00462410`|cdecl() -> window*|the topmost open window|
|`0x00462390`|thiscall|show: append (the base open)|
|`0x00462320`|thiscall|remove, submitting the window's rect first|

The driver has exactly two callers, both scene draw methods - the local map's
(`0x005888FC`) and the scrollmap's (`0x0044AC04`) - which is what makes a window in this
list scene-independent: the ship overview opens on the world map and in a town alike.
Dismissal is centralised on the list too: ESC (`0x004247DA`) and right-click
(`0x004298F0`) call `0x00462230` when `0x00462310` says something is open, and a window
opening closes the others with `0x00462280(0)` first.

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
[frame clock](./time.md#the-frame-clock): neither the game tick nor the
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

## Building Interior Animations
The animated characters inside building windows - the church's priest, the tavern's
sailor, captain, informant, pirate, weapons dealer, burglar and traveller - are all
played by **one shared animation player**: a 0x3C-byte object at `[0x006CC7F4]`,
created in the session UI init (`0x00426B0C`, constructor `0x0046AFB0`) and destroyed
with the rest of the UI (`0x00428234`). It holds one animation at a time and streams
its frames from BMP files under `images/Gebaeude_innen/` (one file per frame, loaded
lazily one frame per call while the animation plays).

|Field|Meaning|
|-|-|
|`+0x4`|the frame-handle array, `malloc(frame_count * 4)` - allocated **only** by switch_animation, `NULL` on a fresh player|
|`+0x8`|next frame due, against the [frame clock](./time.md#the-frame-clock)|
|`+0xC + id`|frame count per animation, byte each, from ini keys read via `0x004BE0B0`|
|`+0x1F`|current frame index|
|`+0x20`|frames loaded so far (the streaming cursor)|
|`+0x21 + id`|frame duration per animation|
|`+0x34`/`+0x36`|draw x/y (u16), set by switch_animation from the window's dimensions|
|`+0x38`|current animation id, `0xFF` = none (the constructor's value)|
|`+0x39`|animation count (0x13)|
|`+0x3A`|playback direction flag (the loops ping-pong)|

|Method|Signature|Role|
|-|-|-|
|`0x0046B710`|thiscall(this, id, window), ret 8|switch_animation: frees the old frames, allocates the array, loads frame 0, positions from the window; if `id` is already current it only resets the timer|
|`0x0046B110`|thiscall(this, id), ret 4|load_next_frame: builds the frame's BMP path and stores the loaded handle at the streaming cursor - **no NULL check on the array**, the crash site of the [church bug](./bugs/church-window-animation-crash.md)|
|`0x0046B530`|thiscall(this, id, a2, force), ret 0xC|tick: advances the frame on the clock and blits|
|`0x0046B6A0`|thiscall(this), ret|draw the current frame|
|`0x0046B0C0`|thiscall(this), ret|teardown: frees frames and array - but leaves `+0x38` stale; the tavern resets it to `0xFF` by hand right after calling it (`0x005CD503`)|

The animation ids, pinned by three independent constraints (the church's
stage-dependent pair selection, the tavern's per-case ids, and the case/string layout
order):

|Id|Animation|
|-|-|
|0 / 1|sailor start / loop (Matrose)|
|2 / 3|informant start / loop|
|4 / 5|pirate start / loop|
|6 / 7|priest loop / thanks, church without altar (`ohneAltar`)|
|8 / 9|priest loop / thanks, church with altar (`mitAltar`)|
|0xA|sailor spits (Matrose_spuckt)|
|0xB / 0xC|captain start / loop|
|0xD / 0xE|weapons dealer start / loop|
|0xF / 0x10|burglar start / loop (Einbrecher)|
|0x11 / 0x12|traveller start / loop (Reisender)|

Only the church and tavern windows drive the player. A byte at
`[[0x006CC3E8]+0x24]` gates the animation paths in both (checked by the church tick
and the player's draw); with it clear, none of this runs.

The interior picture the characters are drawn on, the whitening veil over it, the chains
and the wooden frame belong to a different shared object, the
[building backdrop](./ui/building-backdrop.md).

## Window Titles
`0x00420C70` (arguments: a string object and the window) draws a window's title banner.
It takes the string **by value and destroys it** before returning (`0x00420DBF` runs the
destructor `0x0064F253` on it), so a caller must construct a fresh string for every
call; handing it the same one twice frees it twice and corrupts the heap. It fetches graphic `0x791E` from the resource manager at `0x006DA820`
(`0x004B3DD0`), takes the four dwords of its rect and renders it together with the
text. Pages that show no title simply never call it, which leaves the strip at the
top of the window free - `mod-tavern-details` uses it for table rows.

## Submitting Screen Areas
The renderer repaints **dirty rectangles only**, and every draw is clipped to them: a
draw method begins with `0x004BB7C0(rect*)` (cdecl, returns 0 when the rect misses the
dirty region, in which case the method returns without drawing), so content that
changes without a submitted rect repaints in slivers or not at all.

`0x004B9650` submits a rect. It is thiscall on the window stack singleton
(`ecx = 0x006DA5F0`) with one stack argument, `ret 4`: a pointer to four dwords - left,
top, right, bottom. It returns without doing anything while `[0x006DCB94]` is non-zero,
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

## Fonts

The game has **six** global fonts, held in an array of containers of stride `0xA8` based at
`0x006DCD28`. `0x004BB8F0` (`sgl_SetFont`) takes the font itself, which is the pointer at
`container + 0x98`.

They are `scripts/fonts.ini`'s `Font0` through `Font5`, in order:

|Index|Container|`fonts.ini`|File|Params (w h spacing)|Code references|
|-|-|-|-|-|-|
|0|`0x006DCD28`|`Font0`|`tiepolo_black16.aim`|15 16 0|458|
|1|`0x006DCDD0`|`Font1`|`tiepolo_bold16.aim`|15 16 0|290|
|2|`0x006DCE78`|`Font2`|`tiepolo_black20.aim`|19 20 0|90|
|3|`0x006DCF20`|`Font3`|`elgreco24.aim` (kerning)|26 24 0|60|
|4|`0x006DCFC8`|`Font4`|`tiepolo_bold24.aim`|25 24 0|30|
|5|`0x006DD070`|`Font5`|`elgreco72.aim`|80 75 0|13|

All six are referenced from code, with the counts above; the next slot along is not a font
container. Pairing the container index to the `fonts.ini` index follows from that ordering
and those counts rather than from a decoded loader.

Note **indices 0 and 1 are the same size** - Tiepolo Black against Tiepolo Bold - so the
face the parchment pages use for headings is *heavier* than their body text, not larger.
Index 3 is a different typeface entirely and is what the scrollmap's town names are drawn
in.

The sizes are not free parameters. `Params` declares each font's cell size, but the files
are **`.aim` bitmap atlases** with the glyphs pre-rendered at that size, so raising the
numbers would misread the atlas rather than scale the text. Six sizes is what exists. The
CJK localisations take a second path - `scripts/ttffonts.ini` maps the same `Font0`..`Font5`
onto `msgothic.ttc`, and there the size genuinely is a parameter because the glyphs are
rasterised on demand.

### The Constant Colour Blends

`0x004BB870` (`sgl_SetConstantColor`) takes `0xAARRGGBB`, and **the alpha byte is
honoured**: a fill through `0x004BB430`
(`sgl_FillSolidRect`) at `0x80......` draws as a translucent plate rather than an opaque
one. Every call the executable makes itself passes `0xFF`, so this is only visible to a mod
that tries it - verified in play with a half-alpha plate behind text on the
[ship panel](./ui/ship-panel.md).


Prefer the thunk over the library's own entry. `ddraw_dll.dll` loads at a *preferred* base
- `0x05000000` in a measured run - so an address inside it is only valid while nothing else
has claimed that range; the exe's thunks are immune, because the exe has an empty
relocation table and the loader fills their slots.

## Graphics and Icons
Every drawing call named on this page is a thunk into `ddraw_Dll.dll`, the
[SGL graphics library](./graphics.md) - that page lists which export each thunk
address resolves to.

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
The numeric input boxes (the trading office's amount and price columns, the goods dialog's
amounts) are a subclass of the text widget (constructor `0x004C9790`, vtable `0x00671890`):
constructor `0x0045C110`, vtable `0x0066DBB8`, `0x190` bytes. They cache their displayed
value and text: the setter at `0x0045C930` (thiscall, one argument) clamps the value to the
bounds at `+0x180` (max) / `+0x184` (min), stores it at `+0x188`, flags `+0x18C` dirty and
rewrites the label text. In-place writes to the underlying data are invisible until either
this setter runs or the owning window repopulates.

The box takes the keyboard while it is the scene's focused child (byte `+0x42`, set and
cleared by the focus callbacks - see [Windows and Widgets](./ui/windows-and-widgets.md#input)).
Its key handler, vtable `+0x1C` = `0x0045C300(vk, repeat, flags)`, returns at once unless
`+0x42` is set; otherwise it marks `+0x18C`, translates the virtual key to a character
(`0x004C0F50`), accepts only `'0'`..`'9'` (`0x0045C6C2`) besides backspace and delete,
edits the text at the cursor (`+0x168`) and stores `atoi(text)` in `+0x188`
(`0x0045C908`). Losing focus (`0x0045C230`) strips leading zeros.

The box does not commit anything itself. Its window reads `+0x188`: the trading office's
page update enqueues an order change every frame for a row whose amount or price box is
focused (see [Trading Office Window](./ui/trading-office-window.md#typing-and-the-per-frame-commit)),
so writing a focused box's value through the setter changes the order exactly as typing
would - which is how `mod-trading-qol`'s Q..Y keys reprice one ware.

## String Objects
Several game functions take an MFC-style string object instead of a plain C string: a
single pointer to character data whose header (refcount, allocated size, length) sits
in the 12 bytes before the data. Passing a raw `char*` to such a function crashes -
the callee dereferences the characters as a pointer.

- construct from a C string: `0x0064F2C1` (thiscall(this, char*)); assign:
  `0x0064F390` (thiscall(this, char*))
- destruct: `0x0064F253` (thiscall(this))
- **by-value arguments are destroyed by the callee** (MSVC convention): the title
  renderer `0x00420C70` and the ini loaders such as the scrollbar's `0x00460B70` release
  the string they are handed, so build a fresh one for every call
- `[0x006C7CCC]`/`[0x006C7CD0]` hold the shared empty-string sentinel; a fresh object
  should be initialized to `[0x006C7CD0] + 0xC` so the constructor's release-old-data
  path is a no-op.

The trade route file loader (`0x004D5EE0`, see
[Trade Routes (.rou)](./file-formats/rou.md)) is one such consumer.

## Render Imports
Drawing goes through the game's own render DLL, `ddraw_Dll.dll` (shipped in the
game directory, distinct from the system's `ddraw.dll`). Its exports are resolved
at startup into slots in BSS, and the game calls them through a block of
`jmp [slot]` thunks at `0x004BAEC0`-`0x004BBB30` - the
[graphics library](./graphics.md#how-the-executable-binds-it) page has the table
that says which export each thunk resolves to.

Text is one of them: `0x004BB3F0` is `jmp [0x006DAA04]`, which resolves to
`sgl_DrawText_Rect` (`ddraw_Dll+0xF100`, cdecl, fourth argument the C string).
It begins with `cmp byte [string], 0` - no validity check of any kind - so any
bad string pointer the game passes crashes *inside* `ddraw_Dll`. A crash address
in `ddraw_Dll` therefore usually means bad arguments from game code, not a render
bug; the caller is on the stack right above (see the
[patrol letter crash](./bugs/patrol-letter-crash.md) for a worked example).
