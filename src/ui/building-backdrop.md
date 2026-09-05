# Building Backdrop

Behind every building window - tavern, church, shipyard, trading office and the rest -
sits **one shared object**, the global at `[0x006CC7E0]`: the building's interior picture,
the whitening veil that makes text readable on it, the two chains it hangs from, and the
wooden frame around it. The windows themselves draw only their widgets and text over it.

It is created in the session UI init right after the loading screen objects: allocated
with `0x100` bytes at `0x00426A14`, constructed by `0x00465010` (vtable `0x0066E688`), loaded
from `./scripts/animation.ini`, and sized **451 x 537** through the widget family's size
slot (`push 0x219 / push 0x1C3` at `0x00426A64`). The building windows are sized right
after it, all twelve to **425 x 510** (`0x00426D98` and eleven more `push 0x1FE / push
0x1A9` pairs in the same routine), which is 13 px inside the backdrop on every side.
Nothing changes either size later; `p3-api`'s `BuildingBackdropPtr` is this object.

## The ini

`[Animation]` in `animation.ini` names the shared textures - `BackTexID=31105`
(`images/Gebaeude_innen/pergament.aim`, 425 x 510, the paper the veil is blitted from) and
`KetteTexID=31106` (`kette.aim`, 32 x 32, the chain) - and `[Rahmen]` the frame,
`RahmenID=31002` (`images/AnimationsRahmen.tga`, 53 x 45, eight pieces). `[AnimN]` sections,
one per building, give the interior picture (`BGID`, 425 x 510 each) and the small
animations drawn on it. The texture ids resolve through `textures.ini` like every other
graphic.

|Field|Meaning|
|-|-|
|`+0x94`|array of graphic records, one per `[AnimN]` - the interior pictures; the record's `+0x4` is the renderer handle|
|`+0x9C`|the frame's graphic record (`RahmenID`)|
|`+0xA0`|the chain's graphic record (`KetteTexID`)|
|`+0xA4`..|the source rectangle the picture is blitted from|
|`+0xE8`|the current building: the index into `+0x94`|
|`+0xEC`|u16, the veil's **gradient y** (below)|
|`+0xF8`|the renderer handle of the veil texture (`BackTexID`)|
|`+0xFC`|byte, non-zero skips the flat veil sheet|

## The draw

The draw (vtable `+0x9C` = `0x00465BA0`, `thiscall(context, x, y, z)`, `ret 0x10`) paints,
in this order:

1. **The picture**: the current building's texture, blitted one to one at the object's
   position plus 12 px on both axes (`0x00465C30`..`0x00465CB4`, through the record draw
   `0x004673F0` with the rect at `+0xA4`). Then the `[AnimN]` sub-animations.
2. **The veil**, from the paper texture at `+0xF8`. If the gradient y is non-zero, a
   160-row fade: row `i` (0..159) is blitted at `y + gradient_y + i - 148`, one row tall,
   `width - 26` wide from `x + 12`, in the constant colour `(i << 24) | 0xFFFFFF` - white
   with alpha rising to 159 - skipping rows not below the object's top edge and rows with
   `gradient_y + i < 160` (`0x00465DCE`..`0x00465E5B`). Then, unless `+0xFC` is set, the flat
   sheet: colour `0xA0FFFFFF`, from `(x + 12, y + gradient_y + 12)`, `width - 26` by
   `height - gradient_y - 27`, source offset `(0, gradient_y)` (`0x00465E67`..`0x00465EAD`).
   So above the fade the picture keeps its colours, below it everything sits under 63%
   white. The windows set the gradient y per page - the office window's page switch writes
   440 for one of its pages at `0x005D9F6E` - which is what lets a page keep more of the
   animation visible where it has no text.
3. **The chains**: frame 0 of the chain record, tiled upwards from the object's top edge to
   the top of the town view at `x + piece` and `x + width - 2 * piece` (`0x00465EDA`..
   `0x00465F9A`).
4. **The frame**, from the eight pieces of the frame record: corners 0..3 at the four
   corners (top-left, top-right, bottom-right, bottom-left), the top bar (piece 4) and the
   bottom bar (piece 5) tiled **17 times** from `x + 12` (`0x00466133`, `0x004661B1`), the
   right bar (piece 6) and the left bar (piece 7) tiled **34 times** from `y + 12`
   (`0x004662B2`, `0x00466236`). The counts are constants; they cover the vanilla size
   exactly.

Two consequences for a mod that enlarges the backdrop: the veil blits read past the
425 x 510 paper texture - the width still falls inside the texture's row pitch, but an
object taller than 537 makes the flat sheet read off the end of the buffer and crash in
`ddraw_dll` - so the two blit calls at `0x00465E45` and `0x00465EA8` have to be redirected
to something that repeats or scales the texture; and the frame bars stop short, so the
bars have to be finished by hand. The picture does not scale on its own either, but the
library's stretch blit (see [Graphics Library](../graphics.md)) can draw it scaled over
the game's copy. `mod-trading-qol` does all three for the trading office.

The picture and the veil are drawn 12 px inside the object, the frame bars are 15 px
thick, so the frame overlaps the picture by 3 px on every side; anything painted over the
picture afterwards has to redraw the frame.

## Positioning

The object's rectangle also decides what the town view repaints: `0x004665D0`, `thiscall`
on the object, submits its own rectangle to the renderer (the same path as
[Submitting Screen Areas](../ui.md#submitting-screen-areas)) and resets `+0xFC`. The
building windows call it when they open, before any mod hook runs, so a mod that resizes
the backdrop afterwards has to submit the new rectangle itself or the added area waits for
the next mouse move.

The shared [animation player](../ui.md#building-interior-animations) for the church and
tavern characters is a different object, `[0x006CC7F4]`.
