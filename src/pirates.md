# Pirates
Pirates are not scripted. The `.p2m` interpreter that runs letter and mission scripts
(`0x004ECF64`) has exactly one caller, the letters task, so no bytecode ever touches a
ship. Everything below is native code: a state machine inside the per-tick ships update,
plus a handful of scheduled tasks that maintain the pirate population.

There are two different things called "pirate":

- a **pirate ship** roaming the map, owned by nobody (`field_0_merchant_index` = `0xFF`)
  and flying status `0x12`. These belong to *bands* based at hideouts, and are what this
  chapter is about.
- a **pirate captain**, one of the tavern characters an
  [auto trader record](./auto-traders.md) can be, whom the player can put in command of
  one of his own ships. Such a ship keeps the player's merchant index and its normal
  status `0x0F`; it is marked only by `field_15C_is_pirate`.

## Bands and Hideouts
The ships container holds up to five **band** objects, pointers at `0x006DD7AC`
(container `+0x0C`). World generation (`0x0054A480`) creates `2 * n + 1` of them, where
`n` is the pirate setting byte at `[[0x006CC3E8] + 0x13]` - so one, three or five bands.
Each is a 24-byte heap object (`new` at `0x0064F7B9`, constructor `0x00513720`, seeded by
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
slot when the tick's low byte is `0x6F` (see [Time](./basics/time.md): a day is 256
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

## Pirate Convoys
The [ships tick](./ships.md) keeps three chains, and an at-sea pirate (status `0x12`,
case `0x00507099`) is pulled out of the at-sea chain and given a **convoy record** of its
own through `0x005062F0`. If its health has reached zero it is removed instead. From then
on the pirate is driven by the convoy loop, case `0x00507A65`, once per tick:

1. if the ship is not in the x-sorted neighbour list (`field_A`/`field_C` both `0xFFFF`),
   `0x0050CE50` inserts it;
2. the restraint counter `convoy+0x16` counts down by one and is mirrored onto the acting
   ship's `field_138`;
3. if `convoy+0x14` bit `0x40` is set, the current prey is re-validated and the pirate
   engages;
4. otherwise, **only on every fourth tick**, it looks for prey with `0x0050D9E0` and asks
   `0x00515360` whether it may attack;
5. engaging unlinks the convoy and calls `0x0050BC40`, which sets convoy status `0x14` and
   hands both parties to the [sea battle](./ships/sea-battles.md) subsystem;
6. with no prey it advances along its course (`0x00502110`), and on arrival unlinks and
   removes the ship.

A pirate always gets a convoy record even when it sails alone, because every comparison
below is made fleet against fleet. Packs of one to three ships have been observed.

## Finding Prey
`0x0050D9E0` keeps the current target as long as it is still at sea and inside the chase
radius; only when that fails does it rescan, walking the neighbour list in both
directions and taking the **nearest** candidate inside the acquisition radius. The radii
live on the ships container and are savegame state, not constants:

|Field|Meaning|Observed|
|-|-|-|
|`+0xFA`|acquisition radius, squared|10000, i.e. 100|
|`+0xFC`|chase radius, squared|12100, i.e. 110|
|`+0xFE`|how far the neighbour walk may run in x|100|

Because the current prey is sticky, a pirate will shadow one ship for days while a closer
one sails past untouched.

Three rectangles in `.rdata` are excluded outright - the map's three rivers. The test is
`0x0050E340`, and the boxes are read from `0x00673564` as x0, x1, y0, y1:

|x0|x1|y0|y1|
|-|-|-|-|
|442|658|1521|1691|
|2265|2332|147|453|
|1497|1578|1263|1444|

## The Decision to Attack
`0x00515360` takes the pirate's ship index and decides whether the latched prey may be
attacked. In order:

- **Never its owner.** `field_15D` holds the merchant a pirate belongs to; that merchant's
  ships are skipped, which is what protects a player's fleet from his own hired pirate.
- **Never a protected merchant.** If the prey's owner has bit `0x4` in `merchant+0x8`, the
  ship is skipped. In a live 24-town game that bit is carried by merchants 0..23 - exactly
  one per town, hometown equal to index - and across 27 observed raids not one victim came
  from that group. The same bit also excludes those merchants from the letter that
  operation `0x3E` posts and from the dynamic name registry used by operations `0xB6` and
  `0xB8`, so it marks a static background merchant rather than a piracy rule as such.
- **The restraint counter.** For an AI-owned ship the pirate's `field_138` must be exactly
  `0`; for a **player-owned** ship anything up to `0x900` (nine days) will do. Both
  branches also consult `[merchant + hometown + 0x39C]`, which must reach 2 together with
  the pirate setting; in a live game that byte reads 3..5 for every merchant, so it never
  blocks.
- **The prey must be carrying cargo** (`field_118` greater than zero) and the pirate's own
  convoy must have at least `0x7D0` raw capacity free - one load - to hold the loot.
- **Speed.** With the per-ship speed of `0x00612930`, the prey convoy's slowest ship
  against the pirate's fastest: `19 * prey > 20 * own` refuses. The prey may be about 5%
  faster and no more.
- **Strength.** `7 * prey > 10 * own` refuses, so a pirate attacks while it has at least
  **70%** of the target's strength. Strength is summed over a convoy as
  `sum(crew) + sum(max(crew, artillery))`, using `field_40_crew` and
  `field_120` (see [Ship Artillery](./basics/ship-artillery.md#combat-power)). For a prey
  ship with no convoy the game uses plain `crew + artillery` instead, so putting a lone
  ship into a one-ship convoy changes - and for an unarmed ship doubles - how strong it
  looks.
- Finally the distance: within 20 units it attacks, otherwise it sets course with
  `0x00516840` and keeps closing.

Neither cutlasses (`field_154_cutlasses`) nor the captain nor the ship's health enter this
comparison. Health and the captain's navigation skill act on the *speed* term instead, and
the captain's combat skill is not read here at all.

## The Raid Cycle
Measured across 27 battles in two campaigns, at 256 ticks per day:

|Event|Value|
|-|-|
|A raiding party leaves its hideout with|counter 0 - free to strike at once|
|Every observed battle against an AI ship began at|counter exactly 0|
|Battles against a player's ship began at|0, and at 1956, 2158 and 2254 - 7.6 to 8.8 days|
|A battle lasts|45..116 ticks, a quarter to half a day|
|After a battle the counter is set to|`0xA00`, ten days (`0x0050C043`)|
|...plus a further|`0x700`, seven days, when the pirate also heads home|
|Rest observed|exactly 10.00 or 17.00 days, nine of each, nothing between|
|Raids world-wide with two or three parties active|roughly one a week|

The seven extra days come from `0x0050603A`, which adds them when the pirate is given its
hideout as a destination; that value then reaches the convoy counter because
`0x00501CC1` loads `convoy+0x16` back out of the acting ship's `field_138`.

What sends a pirate home is a fitness check, not a timer. In `0x00505C90` it breaks off
when its health drops below 80% of maximum or its artillery falls under 18 power - two
small catapults' worth (`0x00505D43`). 80% is exactly where the speed term stops
saturating, so a raider retreats at the moment damage starts costing it speed.

The practical consequence of the counter is an asymmetry in the player's disfavour: an AI
merchant's ship can only be taken in the single tick the counter is zero, while a player's
ship is fair game for a nine-day window in every cycle.

## Ship Fields
|Field|Meaning|
|-|-|
|`field_15C_is_pirate`|set by operation `0x0D` (`0x005386C0`) on a ship that is at sea; also ORs `0x18` into `field_3D`. This is the flag a hired pirate's ship carries|
|`field_15D`|the merchant a pirate belongs to, `0xFF` for a free pirate. Written by the ship spawner `0x00509250` from the ship's own merchant index|
|`field_158`|the band index, 0..4. The getter `0x0051A470` falls back to the first surviving band, so an orphaned raider re-homes itself|
|`field_159`|an assigned town index, stored only for ships without a real owner (`0x0051A83B`), otherwise `0xFF`|

`field_158` and `field_159` are late additions: the ship loader only reads them when the
savegame version is at least `0x79`, defaulting them to 0 and `0xFF`.

A pirate ship that puts into a town whose `+0x2C8` lacks flag `0x04000000` is removed
outright (`0x00506DEB`), which is why pirates are only ever seen entering their hideouts.

Capturing a pirate clears `field_15C` and normalises the status, so a prize behaves like
any other ship - it keeps its name, its index and a now-meaningless band number. It
arrives stripped: one captured hull came with 8 crew against a complement of 29, 55% hull,
and an empty two-slot artillery position where a bombard had been shot away.

## Operations and Tasks
|Opcode|Effect|
|-|-|
|`0x0D`|set or clear `field_15C_is_pirate` on a ship at sea (`0x005386C0`)|
|`0xB4`|`0x00542E40`, which reaches the pirate ship creator|
|`0xB5`|create a pirate ship (`0x00514D40`) for the band named by the operation's first argument|
|`0x99`|form or join a convoy (`0x0050B250`) - works for a single ship|

Task `0x08` (`0x004E2634`) is unrelated to these ships: it maintains the tavern
population of captains and pirate captains, and is described under
[Auto Traders](./auto-traders.md).
