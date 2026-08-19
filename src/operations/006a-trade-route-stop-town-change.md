# Trade Route Stop Town Change
Operation `0x6A` inserts a stop into an applied trade route or removes one - the route
panel's town selection, where choosing "none" removes the stop. Stops are identified by
their index in the route stop pool (see [Trade Routes (.rou)](../file-formats/rou.md)),
not by their position in the route.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x6A`|
|0x04|u32|stop pool index (validated against the pool count `[0x006DD72A]`)|
|0x08|u32|insert flag: 0 = remove this stop, 1 = insert a new stop after it|
|0x0C|u32|town index; `0xFF` ("none") on removal|
|0x10|u32|ship index|

The handler at `0x0053E610` (operation switch case `0x5364E3`):

- **Insert** (`+0x08` set): validates the town, allocates a fresh pool record through
  the pool allocator (`0x004D4C90`, `this = 0x006DD728`), links it into the chain after
  the record at `+0x04` and sets its town. The record's instructions start empty.
- **Remove** (`+0x08` zero, town `0xFF`): moves the first-stop marker (action bit
  `0x04`) to the successor if the removed stop carried it, retargets ships heading for
  the removed stop (`0x00509030`), and releases the record through the pool free
  (`0x004D4E80`).

Freed pool records are reused by a freelist: removing a stop and adding a new one
hands out the same index again (verified in-game), which is why stop identity must
always be taken from the live chain.
