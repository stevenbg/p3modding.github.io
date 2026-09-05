# Graphics Library (SGL)

All rendering goes through `ddraw_Dll.dll`, Ascaron's own graphics library - its
debug path names the project: `D:\coding\SGL_DDRAW\Release\ddraw_dll.pdb`. It
exports 198 functions, all prefixed `sgl_`, and does its own software drawing on
top of DirectDraw surfaces.

It must not be confused with `ddraw.dll` in the same folder: that file exports
`DirectDrawCreate`, `DDInternalLock`, `D3DParseUnknownCommand` and the rest of the
Microsoft DirectDraw API and is GOG's replacement for the system component. It
knows nothing about the game.

## How the Executable Binds It

The executable does not import `ddraw_Dll.dll` statically. It carries a table of
180 entries at `0x006BBC2C`, one per function it wants, each `0x44` bytes:

|Offset|Size|Meaning|
|-|-|-|
|`-0x04`|4|pointer to the slot that receives the resolved address|
|`+0x00`|`0x40`|the export name, NUL terminated (`sgl_DrawBitmapRect`, ...)|

Note that the slot pointer sits **in front of** the name it belongs to. The resolver
(`0x004BAD38`) keeps `esi` on the name, passes it to `GetProcAddress`, and stores the result
through `eax = [esi-4]` (`0x004BAD67`) before stepping `esi` by `0x44`. So an entry's own
`+0x40` dword is the *next* entry's slot pointer, and reading it as this entry's - even with
the `+4` the mostly-descending pointer run seems to invite - mis-attributes any entry where
that run is not monotonic. `sgl_SetTextMode` and `sgl_SetYAxis` are such a pair.

Every bound function also gets a thunk of the form `jmp DWORD PTR [slot]`, usually 16 bytes
apart, in the `0x004BAEC0`-`0x004BBB40` block, and that thunk address is what the rest of the
executable calls. This is how the drawing helpers used elsewhere in this book resolve:

|Thunk|Export|
|-|-|
|`0x004BB140`|`sgl_AddClipRect_r`|
|`0x004BB330`|`sgl_DrawBitmapRect`|
|`0x004BB340`|`sgl_DrawBitmapRectWithAlphaMask`|
|`0x004BB430`|`sgl_FillSolidRect`|
|`0x004BB4A0`|`sgl_FreeTexture`|
|`0x004BB490`|`sgl_FreeMemoryTexture`|
|`0x004BB620`|`sgl_LoadTexture`|
|`0x004BB640`|`sgl_LoadTextureImage`|
|`0x004BB650`|`sgl_FreeTextureImage`|
|`0x004BB780`|`sgl_SetActiveClipper`|
|`0x004BB870`|`sgl_SetConstantColor`|
|`0x004BB8F0`|`sgl_SetFont`|
|`0x004BB9B0`|`sgl_SetRenderDest`|
|`0x004BB9C0`|`sgl_SetRenderSource`|
|`0x004BBA10`|`sgl_SetTextMode`|
|`0x004BBA80`|`sgl_StretchBitmapRect`|
|`0x004BBB20`|`sgl_GetTextureInfo`|

Reading the table is the reliable way to name any of the ~180 drawing calls in
the executable, and the same walk recovers the slot a mod can hook to intercept
one (see [Graphics and Icons](./ui.md#graphics-and-icons) for the blit sequence
these are used in).

### Blits Do Not Clip to the Texture

`sgl_DrawBitmapRect(src_x, src_y, x, y, width, height)` copies exactly the rectangle it
is given out of the selected texture, one pixel to one. It does not check the source
against the texture's size: a rectangle wider than the texture still lands inside the
row pitch and merely shows the padding, but one that runs below the last row reads off
the end of the pixel buffer and crashes inside `ddraw_Dll` (the
[backdrop](./ui/building-backdrop.md)'s veil does this when the backdrop is made taller
than 537). A caller that needs a larger area either repeats the texture in pieces or uses
the scaled blit.

`sgl_StretchBitmapRect` takes eight arguments, in this order:
`(src_x, src_y, dst_x, dst_y, src_width, src_height, dst_width, dst_height)`, and scales
the source rectangle onto the destination one, modulated by the constant colour like the
plain blit. The order comes from the game's own uses: the minimap builder at `0x004B1463`
copies a whole measured texture into a memory texture of a different size as
`(0, 0, 0, 0, tex_w, tex_h, dst_w, dst_h)`, and `0x004B27B5` stretches from `(0, 0)` onto
a destination rectangle. The library also exports what a mod would need to build textures
of its own - `sgl_CreateMemoryTexture`, `sgl_LoadMemoryTexture` and `sgl_LockTexture` are
bound by the executable too (slots `0x006DAA4C`, `0x006DA990`, `0x006DA85C`), and
`sgl_UnlockTexture` is exported; their signatures are not traced.

## The Decoded Image Cache

Sprites are drawn from decoded source images, and those images are kept in an
LRU cache inside the library. Everything about it is internal - none of it is
visible through the `sgl_*` exports, which is why hooking the library's free
functions never shows a cache eviction.

|What|Where|
|-|-|
|Bytes currently held|`ddraw_Dll+0x80F14`|
|Getter / setter|`ddraw_Dll+0x1C1D0` / `+0x1C1E0`, thiscall on the object at `ddraw_Dll+0x6E960`|
|Budget|`ddraw_Dll+0x5F734`, default `0x01000000` (16 MiB)|
|Eviction loops|`ddraw_Dll+0x1C2EF` and `+0x1C39A`|
|Load (decode) path|`ddraw_Dll+0x13350`, reached through the gate at `+0x130A0`|

Both eviction loops read the counter, compare it against the budget, and while
it is greater release the least recently used entry through its vtable `+0x20`.
The release path ends at `ddraw_Dll+0x1309B`: it zeroes the image's `+0x18` and
`+0x1C`, frees the pixel allocation at `+0x44`, and subtracts that allocation's
size from the counter - `WORD [img+0x22] * DWORD [img+0x28]`, the same product
the load path adds, which is why the counter is a byte total and the default
budget is exactly 16 MiB.

The gate in front of the loader decides per draw whether a decode is needed:

- if the texture has an image (`[tex+0x40]`) and that image still has its pixels
  (`[img+0x18]`, the field the release path zeroes), the draw proceeds from the
  cache;
- otherwise, if the state byte `[tex+0x3D]` is 5, the image is decoded from the
  memory file at `[tex+0x2C]` with the flags at `[tex+0x34]` or-ed with
  `0x30000`.

The decode itself is handed to `AIM.dll` (see [AIM Images](./file-formats/aim.md)),
which `ddraw_Dll.dll` imports: `AIM_CONVERT_MEMFILE` at IAT `+0x50028` and
`AIM_CONVERT_RAW` at `+0x50034` do the work, `AIM_INIT` and `AIM_FREE` bracket
them. Only `ddraw_Dll.dll` and `Vto.dll` import `AIM.dll`; the executable never
calls it directly.

Because the budget is a hard ceiling with least-recently-used eviction, a working
set slightly larger than the budget degenerates into a complete miss rate. That
is a real, shipped bug - see
[Texture Cache Thrash](./bugs/texture-cache-thrash.md).

## gl.cfg

`gl.cfg` in the game folder is the library's configuration file - and exactly one
key in it still works.

**The live key is `DLL`, and the executable reads it.** During startup
(`0x004BC010`, called from `0x0046359E` before `game.ini` is processed) the game
opens `gl.cfg`, reads it line by line, splits each line on `" \t=\n"` and
compares the first token against `DLL`, case insensitively. The second token of
a matching line becomes the name of the render DLL to load; with no such line the
default `"ddraw_dll.dll"` (`[0x006BEEC4]`) is used. No section header is
required - the reader scans every line. This is the supported way to point the
game at a different or wrapped renderer.

**Every key belonging to the library itself is dead.**
`P3HardwareSettings.exe` writes the file with a `BEGIN D3D` section and the keys
`NoMMX`, `NoISSE` and `NoHardwareScroll`, and `ddraw_Dll.dll` contains a parser
for those plus `VideoMemorySize`, `TextureCacheSize`, `ScreenWidth` and
`ScreenHeight` (key dispatch from `ddraw_Dll+0x11E65`, value parser `+0x11810`,
section scanner `+0x12190`, filename string `+0x5FA40`). None of it runs: the
section scanner has no callers anywhere in the library, and nothing references
the filename string, so the library never opens the file and none of its own
keys reach the running game.

That distinction matters in practice. GOG's shipped `gl.cfg` sets
`TextureCacheSize = 48000000`, which would have been large enough to avoid
[the cache thrash](./bugs/texture-cache-thrash.md) entirely - and it has never
had any effect. The v1.1b beta patch does not change this either: it ships only
a `Patrician3.exe` and a CRT, no replacement library, and its executable carries
the same literals (`SGL`, `gl.cfg`, `DLL`, `PatternFile`, `Adapter`, `Size`,
`ColDepth`) with no cache keys at all.

## Scene Sprites

The town and sea maps draw from a global array of graphic definitions at
`[0x006E2E70]`, with its entry count at `[0x006E2E6C]`. Definitions are indexed
by a byte, and each one holds:

|Offset|Meaning|
|-|-|
|`+0x14`|frame count|
|`+0x18`|frame table, 12 bytes per frame (`+4` width, `+6` height, `+8`/`+0xA` draw offsets)|
|`+0x34`|the texture handle passed to `sgl_SetRenderSource`|

A parallel byte table at `[0x006E2E74]` holds per-definition flags; bit `0x40`
gates whether the definition draws at all.

The scene window ([the local map scene](./ui.md#the-local-map-scene),
`[0x006E51AC]`) owns the objects that use them: its tile map is at `+0xC310` with
the row stride at `+0xC314` and the row count at `+0xC318`, holding `WORD` ids that
index the object table at `+0x3FC`, and the current scroll offset sits at `+0xC650`/`+0xC654`. A sprite
object carries

|Offset|Meaning|
|-|-|
|`+0x0E` / `+0x12`|world x / y|
|`+0x14`|animation phase|
|`+0x15`|frame selector|
|`+0x2C`|graphic definition index|
|`+0x30`|alpha, `0xFF` for opaque|

and draws through its vtable: for the most common tile class (vtable
`0x00676398`) the draw slots `+0x8` and `+0xC` are `0x00579B80` and `0x00579EC0`.
Both select the definition's texture with `sgl_SetRenderSource`, apply the alpha
through `sgl_SetConstantColor` when `+0x30` is not `0xFF`, and blit with
`sgl_DrawBitmapRect`. Linked graphics are dispatched to the object table through
`0x005599D0`, whose tail jumps to the target object's own vtable `+0x8`.
