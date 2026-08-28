# Won Game Record (.eld)
`LastWon.eld` sits loose in the game folder and is the only `.eld` file P3 produces. It holds
a **serialised merchant**, packed with the same compressor the
[trade route files](./rou.md) use.

Only one routine touches it, and only to write it: the filename appears exactly once in the
executable, at the `CFile` open in `0x005499F0`, with mode `0x9001` -
`typeBinary | modeCreate | modeWrite`. Nothing loads it back under that name.

## Container
Identical to [`.rou`](./rou.md): a 4-byte output length followed by the bit-packed stream, and
a **negative** length marking a payload stored verbatim. `p3-rou`'s decompressor reads it
without modification - the shipped `LastWon.eld` is 4992 bytes on disk, declares `0x2920`, and
unpacks to exactly 10528.

## Payload
`0x005499F0` resolves the player merchant from `operations + 0x924` and serialises it:

|Step|Routine|
|-|-|
|serialised size|`0x004F3FE0` - `0x004F6CE0(merchant) + 0xE4`, plus ten `u16` at `merchant + 0x60C` (masked `0x7FFF`) counted twice each, so ten variable-length tails|
|write the merchant|`0x004F4010(merchant, buffer + 4)`, past the 4-byte length prefix|
|write a second object|`0x0054CBD0`, sized by `0x0054CB90` - only when that object is present|

The decompressed bytes carry long runs of `0xCD`, so some of what is serialised is
uninitialised heap rather than meaningful state. The [merchant struct](../merchants.md) is
`0x650` bytes, well under the 10528 written, so most of the payload is those variable-length
tails and the second object.

## When it is written
[Scheduled task `0x20`](../scheduled-tasks.md) (`0x004DBBD0`) is the sole caller. Immediately
before writing it calls `0x00469AF0` with **`0x25`** - the highest of the `0x26` scene ids
that routine takes (a `0x25` special case at `0x00469BC3`, then `cmp ebx,0x24 / ja` bails at
`0x00469C06`) - so the file is produced as the game switches to that screen. What schedules
task `0x20` has not been identified, so the exact trigger is unconfirmed; the filename and the
scene are the only evidence that it is a won game.
