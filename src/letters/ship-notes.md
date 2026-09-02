# Ship and Route Notes

The one-line ship messages on the ticker - "The cog %s has docked in %s.", "%s's trade
route: crew number too low" - are notes filed through a single creator and rendered by
a single dispatcher. Knowing the subtype space is what lets a mod intercept one message
without touching its neighbours.

## The creator: `0x00548CA0`

A thiscall on the operations object (`0x006DF2F0`) with six stack arguments
(`ret 0x18`):

```c
create_note(merchant, 0, 0, town, subtype, ship_index);
```

The ship index is passed as a dword but only the low word is meaningful - callers push
registers with stale upper bits (measured: `0x006D007D` for ship `0x7D`), so a consumer
reading the argument must mask it.

## The renderer

The note's subtype (0 .. `0x2A`) goes through a two-level dispatch: the byte case map
at `0x00548834` picks one of 19 cases, and the jump table at `0x005487E8` lands in the
case body (the bounds check and indexed jump are at `0x005483F6`..`0x0054840B`). The
trade-route texts live in the pointer table at `0x006B04F4`, indexed 6..11; their case
bodies build the index as staggered constants (five entry points into one instruction
run at `0x005486CF`..`0x005486D9`).

The subtypes pinned so far (extracted from the case map, message texts from the format
table):

| subtype | message |
|-|-|
| `0x05` | "The cog %s has docked in %s." (the arrival note; format via the case at `0x548540`) |
| `0x22` | "%s's trade route is interrupted" |
| `0x23` | "%s's trade route: ship condition too poor" |
| `0x24`, `0x25` | "%s's trade route: no destination specified" |
| `0x28` | "%s's trade route: crew number too low" |

The `%s` is the ship's inline name (`ship+0x160`).

## Known creation sites

- **`0x00519D72`** (subtype 5, inside the dock function `0x00519C90`): filed when a
  ship docks, guarded by: human owner, ship not in a convoy, route bit
  (`ship+0x136 & 1`) clear.
- **`0x00518AB2`** (subtype `0x28`): the trade-route departure check at `0x00518A96`
  compares the crew (`ship+0x40`) against the per-type minimum (`0x673660`, see
  [Crew](../ships/crew.md)); a crew below it files the crew-number-too-low note, one at
  or above it branches to `0x00518ABD` and files subtype 0 instead.
