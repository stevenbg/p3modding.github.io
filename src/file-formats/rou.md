# Trade Routes (.rou)
The saved trade routes are stored in `./Save/AutoRoute`.
An uncomporessed trade route is an array of route stops, 220 bytes each.
The rou file format is defined as:
```
          | 00  01   02  03   04  05   06  07 |
00000000  | Output Length   | Data            |
```

If the output length is bigger than `0` the file is compressed.
Otherwise the absolute value of the negative output length denotes the length.

## Compression
The compression algorithm has not been identified, but the decompression was [reproduced](https://github.com/P3Modding/p3-lib/tree/master/p3-rou).

## Trade Route Stops
A trade route stop is defined as:
```
          |           00           01           02           03           |
00000000  | Unused                        | Town Index | Action           |
00000004  | Ware Order Array                                              |
...
0000001c  | Ware Price Array                                              |
...
0000007c  | Ware Amount Array                                             |
```
The "direction" of a transaction is encoded in the price and amount:

|Price|Amount|Direction|
||||
|0|Negative|Ship -> Office|
|0|Positive|Office -> Ship|
|Positive|Positive|Ship -> Town|
|Negative|Positive|Town -> Ship|

The "Max" amount is represented by `1_000_000_000` for both barrel and bundle wares.
Amounts are stored in raw units: display units times the ware scaling (bundles 2000, barrels 200).

The ware order array is a **sequence of ware indices**, not a set of flags: it is the order
in which the stop's instructions are carried out. Entries outside `0..0x17` are skipped
rather than ending the sequence, and a ware whose amount is `0` carries no instruction. The
order only sequences wares within each of the two passes the executor makes over it - see
[Running a Route Stop](../ships/trade-routes.md) for what actually happens
at a stop.

## Action Byte
The action byte combines the stop's repair flag with a first-stop marker:

|Value|Meaning|
|-|-|
|0x00|repair setting "X"|
|0x01|repair setting "R" (repair at this stop)|
|0x09|repair setting "-"|
|0x04|OR'ed onto the route's logical first stop|

## Applied Routes at Runtime
Loaded routes live in a global pool of the same 220-byte stop records, prefixed by a
2-byte next-stop index in the record's first two ("Unused") bytes:

- `[0x006DD72C]` = pool base, `[0x006DD72A]` (u16) = pool record count.
- A route is a circular chain of records through the next-stop indices; the stop
  carrying action bit `0x04` is the logical first stop.
- `ship+0x132` (u16) = the pool index of the ship's current route stop; it advances as
  the route runs.

## Loading Path
The game loads a route file through the loader at `0x004D5EE0` (thiscall,
`this = 0x006DD728`): it takes a pointer to an MFC-style string object holding the base
name and forms the path `save\AutoRoute\<name>.rou` itself, returning the decompressed
stop buffer. To attach the route to a ship, `transfer_loaded_traderoute` (`0x005492D0`,
thiscall on the operations struct `0x006DF2F0`) reads the buffer pointer from
`operations+0x930` and the target ship index from `operations+0x934`, validates the
stops, allocates pool records, attaches them to the ship's convoy, and frees the buffer
with the game's own allocator.
