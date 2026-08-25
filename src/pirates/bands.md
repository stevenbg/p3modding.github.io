# Bands and Hideouts
The ships container holds up to five **band** objects, pointers at `0x006DD7AC`
(container `+0x0C`). World generation (`0x0054A480`) creates `2 * n + 1` of them
(`lea ebx,[edx+edx*1+0x1]` at `0x0054A4F8`), where `n` is the **Pirates activity** setting:
the byte at `[[0x006CC3E8] + 0x13]`, holding **0 for low, 1 for normal and 2 for high**. The
[Game settings](../reference/game-settings.md) steppers are 1-based on screen and stored one
lower (`0x0049935B` copies it from `[window+0x1B90]`). That gives one, three or five bands,
and five is why the game reserves exactly five band slots.

"Difficulty" is only a preset over the individual settings, and this is one of the seven bytes
it moves; where the presets are expanded into them has not been found. Every reader of the
byte other than the settings screen is pirate code, so it governs nothing else.

Each band is a 24-byte heap object (`new` at `0x0064F7B9`, constructor `0x00513720`, seeded by
`0x00513D80`):

|Offset|Meaning|
|-|-|
|`+0x0`, `+0x4`|lazily created sub-objects (`0x00514530` allocates a 0x6C-byte one)|
|`+0xA`|the convoy index the band's raiding party uses|
|`+0xE`|hideout index|
|`+0xF`|a class byte, `1` when the hideout's own class is 3..5, otherwise a random 0/2/3|
|`+0x12`|head of the band's ship chain (ships linked through `field_6_next_ship_index_in_convoy`)|
|`+0x14`|behaviour class, `rand & 3`|
|`+0x15`|state; `1` makes `0x00514000` return the ships, free the object and null the slot|

Every band is ticked once per in-game day - the ships tick calls `0x00514000` on each
slot when the tick's low byte is `0x6F` (see [Time](../time.md): a day is 256
ticks). Several scheduled tasks also work on these objects: opcodes `0x1F`, `0x23`,
`0x24`, `0x25`, `0x30`, `0x32` and `0x34`.

Hideouts come from a runtime table at `0x006DDBB0`, 52 bytes per record, at least 48 of
them readable:

|Offset|Meaning|
|-|-|
|`+0x0`|x (read at `0x00505FD5` and `0x0051516D`)|
|`+0x4`|y|
|`+0x8`|pointer into a 0x50-stride array|
|`+0x10`|region id, 0..3|
|`+0x11`|class byte 0..7, which drives the band's `+0xF`|
|`+0x12`|`0x2D` followed by `l`, `r` or `c` - ASCII, so part of a name|
|`+0x14`..|four (x, y) dword pairs close to the hideout|

A hideout is not a place on the map: a ship that reaches its hideout's coordinates is
taken off the map entirely (`0x0050D040`, then `0x00514B80` hands it to the band) and
parks at position `32767, 32767`. While parked it is **repaired at exactly 1000 hull per
day** - measured over 61 days on one ship and 41 days on another, which was built from
nothing at the same rate - and it is also refitted: artillery totals climb back to the
hull's full fit, while crew losses are made good more slowly. `0x00514D40` dispatches a
ship again only when its captain is a pirate record and its health is back at maximum
(`0x00514DE0`).

`0x00514B80`, the hand-over itself, walks the arriving convoy and for each ship unlinks it
from the convoy, links it into the band's chain at `+0x12`, parks it off the map, and:

- **cashes the cargo into the band.** Every ware is zeroed and its amount scaled by the
  per-ware factors at `0x00673A18`; the total divided by 1000 is added to the band's
  `+0x16`.
- **awards the captain.** For every ship that carries one, it enqueues
  [operation `0x12`](../operations/0012-auto-trader-skill-gain.md) with a gain of `50` for
  navigation and `50` for the other skills (`0x00514C93`), crediting the **acting** ship of
  the convoy (`convoy+0x10`) rather than the ship being processed - so a multi-ship raiding
  party pays its leader once per crewed ship. This is the fastest skill growth in the game;
  see [Skill](../auto-traders/skill.md).
- **upgrades a ship below upgrade level 2** through `0x0051A750(ship, 1)`.
- **drops the owner link of a badly damaged ship.** At `0x00514C5F`, a ship arriving with
  less than half its maximum hull has `field_15D` set to `0xFF`. Since that field is what
  marks a raider as somebody's hired pirate, a privateer that limps home stops being its
  owner's: it is repaired at the hideout and put back to sea as a free pirate, with nothing
  in the interface to say so.

