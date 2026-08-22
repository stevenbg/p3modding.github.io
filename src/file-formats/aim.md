# AIM Images (.aim)
The engine's image format, handled by `AIM.dll` and selected as
`ShaderFormat=aim` in `iso.ini`. It is used both for tile bitmaps in
`./images/module_stadtkarte/` and for baked town overview images in
`./iso/towns/<id>.aim`.

The file starts with the magic `AIM\0`, followed by an embedded name string and
a sequence of chunks of the form `{ Type, Size, Data }`:
```
0000  41 49 4d 00 ...        "AIM\0"
0009  6c 61 6c 61 00         embedded name string
...   chunks { Type, Size, Data }
```

| Chunk Type | Encoding | Example |
|-|-|-|
| `22` | RGB / BGR, uncompressed | town overview `0.aim` (`272 x 155`) |
| `34` | compressed indexed BGRA | tile bitmaps |

Tile bitmaps are sheets of `34 x 21` diamonds keyed on magenta (`FF 00 FF`); a
single file may hold several variants. The overview `iso/towns/<id>.aim` decodes
to a `272 x 155` painted image (the same size as the matching `.tga`), not a tile
render.

`.aim` files can be converted to and from PNG with the community `aim_converter`
tool (`to-png` / `to-aim`).

## The Codec
`AIM.dll` exports the codec as 32 decorated C++ symbols - `AIM_INIT`,
`AIM_CONVERT_FILE`, `AIM_CONVERT_MEMFILE`, `AIM_CONVERT_RAW`, `AIM_FREE`,
`AIM_WRITE_IMAGE` and so on - operating on an `AIM_IMAGE` struct whose first
fields are the pixel pointer, a second buffer pointer, width and height. Only
`ddraw_Dll.dll` and `Vto.dll` import it; the executable never calls it directly,
so every decode in the game is one the
[graphics library](../graphics.md#the-decoded-image-cache) asked for.

Inside the DLL, every file decode passes through one call site at `AIM+0x2984`,
a cdecl function taking `(image, file_path, file_data, file_size)`. The path
argument makes that site the place to observe or substitute images by name -
[mod-high-res](../patches/high-res.md) hooks it to swap in larger background
art, and it is what identifies the files behind the
[texture cache thrash](../bugs/texture-cache-thrash.md).

Decoding is pure CPU work and not cheap: measured against the game's own
`AIM.dll`, chunk-`34` strips decode at roughly 180-250 Mpx/s on a modern
machine, so the shipyard's `362 x 4620` water strip costs about 9 ms and the
trading office's `7740 x 363` overlay about 11 ms per decode. JPEG assets
(`innenbild_werft01.jpg` and friends) go through `ijl11.dll`, the Intel JPEG
Library, at comparable cost.

[To be completed]