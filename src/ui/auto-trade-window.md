# Auto Trade Window

The route window - the one the ship panel's "Auto trade" button opens - draws the route's
stops into a fixed bank of **exactly 20 row widgets**, built once when the window is
constructed (`mov eax,0x14` bounding the loop at `0x00489E07`). The rows are two parallel
arrays embedded in the window object, with no slack after either:

|Array|Start|Stride|Ends at|
|-|-|-|-|
|A|`+0x2340`|`0xF8`|`0x36A0`, where the next member begins|
|B|`+0x589C`|`0xE8`|`0x6ABC`, where the next member begins|

## The 20-stop limit, and the crash past it

The populate routine (`0x004934B0`, single caller `0x0048C288` - it runs on open and on
route change, not on scroll) walks the route's stop chain and **never checks the row
count**: its loop (`0x00493705`ff) only terminates when the chain wraps to its first stop.
Stop 21 therefore writes past array B's end into unconstructed memory, and the `CString`
assignment at `0x004C7780` dereferences a NULL `m_pchData` - an access violation at
`0x0064F234` reading `0xFFFFFFF4`.

Vanilla cannot reach this: the window clamps its own "add a stop" row to 20 at
`0x0049379D` (`cmp eax,0x14`). But the [stop pool](../file-formats/rou.md) holds **150**
records (`[0x006DD72C]`, count `[0x006DD72A]`), so a longer route is perfectly legal
*data* - a route built by a mod or loaded from a crafted `.rou` file crashes the game the
moment this window opens on it.

## The scrollbar is a viewport, not a list

The scrollbar appears once the route holds more than 5 stops (`cmp ebx,5 / setg` at
`0x004937BB`) and moves the view over the 20 pre-built widgets. It is not virtualised:
scrolling does not re-run the populate, which takes no scroll offset, and the interaction
handlers index the row arrays directly by row position (e.g. `0x0048C58F` reading
`[edx+esi+0x6ABC]`). Raising the limit would therefore mean either growing the window
object - shifting some 200 hardcoded member offsets that live above the arrays - or
virtualising the rows and remapping every handler.
