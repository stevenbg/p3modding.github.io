# Image Widget

`C2DAnimation` (name at `[0x006BB3D8]`) is the game's picture: one texture frame, or a
timed sequence of frames, drawn as a widget. The four looks of a [button](./buttons.md)
are objects of this class, and so is the checkmark of the trading office's lock checkbox.
`p3-api`'s `ui::animation` wraps it.

Constructor `0x004ACDD0` (thiscall, no arguments), vtable `0x006702F0`, `0xD8` bytes;
the frame sequence object sits at `+0xA8`. **`0x004ACE40` is the destructor** (slot 0
`0x004ADC50` calls it).

|Function|Signature|What|
|-|-|-|
|vtable `+0x8` = `0x004ACF90`|thiscall(CString by value, id), `ret 8`|load `[ANIM<id>]` (`0x006BB2FC`) from the ini: `Count`, `FrameCount<n>`, `Frame<n>` entries read as `TexID frame x y _ w h _` (`%d %d %d %d %*d %d %d %*d`), and the timers. The string is destroyed by the callee|
|`0x004ADA30`|thiscall(this = source, dest), `ret 4`|copy: the widget base, the sequence and the frame state|
|`0x004ACEA0`|thiscall()|reset|
|vtable `+0x18` = `0x004AD890`|thiscall(point*, type), `ret 8`|the event handler: **a bare `ret 8`**|

The empty event handler is what makes the class stackable: a window can register an image
right after a button, so it draws on top, without taking the button's clicks. The
trading office's lock checkmark is `[ANIM12]` of `./scripts/BuildingParchment.ini`
(`Frame0=20008 0 0 0 0 24 16 0`: TexID 20008, frame 0, 24 x 16), placed at
`(button.x - 4, button.y)` over a 16 x 18 round button and shown or hidden to mean locked
or not.
