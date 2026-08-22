# Texture Cache Thrash

## Summary
Opening a building window collapses the framerate: the game drops from a locked
48 fps to around 17-20 fps and recovers the instant the window is closed. It is
worst in the shipyard, clearly noticeable in the bath house, and absent in the
tavern and town hall. The drop happens in the unmodded game, at any resolution,
and even on a page that draws nothing at all - it does not depend on what the
window shows.

While the window is open the process spends roughly **800 ms of every second**
inside `AIM.dll`, decoding images. The files being decoded are not just the
window's own art: the coastal cliffs, the harbour water, ship sprites, town
walkers and the building's background photo all re-decode from the archives
dozens of times per second, every frame, for as long as the window is open.

## Details
Sprites are drawn from decoded source images, which the graphics library keeps in
an LRU cache with a hard byte budget - see
[the Decoded Image Cache](../graphics.md#the-decoded-image-cache) for the
addresses. The budget's built-in default is **16 MiB**
(`ddraw_Dll+0x5F734` = `0x01000000`).

A town view alone fits comfortably. A town view *plus* an open building window
does not: measured with a live counter on `ddraw_Dll+0x80F14`, the working set
settles at about **19.4 MB**. That is only ~3 MB over the ceiling, but the
consequence is total: the eviction loop drops the least recently used images to
get back under budget, and those are precisely the images the next frame draws
again. Every frame evicts what the next frame needs, so the hit rate collapses to
zero and the whole visible scene is re-decoded continuously.

This also explains the symptoms that look inconsistent:

- **The shipyard is the worst** because its interior has the largest animated
  overlays (a `362 x 4620` water strip, seagulls, a pulley, workers), so it pushes
  the working set furthest past the ceiling.
- **The drop deepens over several seconds** rather than appearing at once, as the
  cache walks its way into the fully thrashing state.
- **The next building window seems slower too** if opened immediately afterwards,
  and normal again after a few seconds in the town view - the cache refilling.
- **The tavern and town hall are fine**: their overlays are small enough that the
  total stays under the budget.

Raising the budget removes the effect completely: the counter settles at the
19.4 MB the scene actually needs, decoding drops from ~800 ms/s to ~10 ms/s, and
the framerate returns to 48 fps within a second.

The library has a configuration key for exactly this - `TextureCacheSize` in
`gl.cfg` - and GOG even ships it set to `48000000`, which would have been ample.
It has never done anything: the library's own
[gl.cfg parser is dead code](../graphics.md#glcfg), never called, so the game
always runs on the 16 MiB default. The v1.1b beta patch does not help either -
it replaces only the executable, which knows nothing about the cache.

## Fix
[mod-fix-texture-cache-thrash](https://github.com/P3Modding/p3-lib/tree/master/mod-fix-texture-cache-thrash)
waits for `ddraw_Dll.dll` to be loaded and writes 48 MiB into the budget at
`ddraw_Dll+0x5F734` - about 2.5x the measured working set, and the same ballpark
as the `48000000` GOG tried to configure. The counter only ever grows to what is
actually in use, so real memory use rises by a few dozen MB at most. The write
happens only if the `0x01000000` default is found, leaving other builds of the
library untouched.

A much larger budget is not better: a display mode switch - alt+tab, or opening
the menu at its own resolution - releases and rebuilds the cached surfaces, so an
oversized cache makes those switches slower.
