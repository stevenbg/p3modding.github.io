# The Pirate AI
A pirate ship at sea is driven by the ships tick, not by a script. This page follows one
raider through a full cycle: how it is given a convoy of its own, how it picks a target,
what makes it decide to attack, and what it does between raids.

## Pirate Convoys
The [ships tick](../ships.md) keeps three chains, and an at-sea pirate (status `0x12`,
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
   hands both parties to the [sea battle](../ships/sea-battles.md) subsystem;
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
`0x0050E340`, and the boxes start at `0x00673564`, 8 bytes each as four u16 in the order
x0, y0, x1, y1 - listed here column-wise:

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
From here the test **splits on who owns the prey**, on the control word `merchant + 0x8`:
zero is a human-controlled merchant, non-zero an AI one. (The same word builds the ten-day
captain scan's own player test with `cmp / sete` at `0x004DCF7E`, and gates the AI
auto-routing call at `0x00506826`, which runs only when it is non-zero.) The two branches
share nothing but the checks further down:

- **AI-owned prey** must not be a **protected merchant**: if the owner has bit `0x4` in
  `merchant+0x8` the ship is skipped. In a live 24-town game that bit is carried by
  merchants 0..23 - exactly one per town, hometown equal to index - and across 27 observed
  raids not one victim came from that group. The same bit also excludes those merchants from
  the letter that operation `0x3E` posts and from the dynamic name registry used by
  operations `0xB6` and `0xB8`, so it marks a static background merchant rather than a
  piracy rule as such. Its restraint counter `field_138` must then be exactly `0`.
- **Player-owned prey** is instead measured by **rank**: the owner's rank in his **home
  town** plus the Pirates activity setting must reach 2 (`0x0051543B`). At *high* activity
  any rank satisfies it, at *normal* it needs rank 1, at *low* rank 2 - so a beginner can be
  beneath a pirate's notice, and the further the setting is turned down the more established
  the player has to be before he is worth attacking. Being successful is what draws pirates,
  which is the opposite direction from an underworld reputation. Its restraint counter is
  the lenient one: anything up to `0x900` (nine days) will do.

  Rank is the byte at `merchant + 0x39C + town`, computed in front of
  [update_merchant_reputation_and_value](../merchants/reputation.md) from the per-town
  reputation float at `merchant + 0x2FC + town*4` and the company value at `+0x46C`
  (see [Ranks](../merchants/ranks.md#where-the-rank-is-stored)). 
- **The prey must be carrying cargo** (`field_118` greater than zero) and the pirate's own
  convoy must have at least `0x7D0` raw capacity free - one load - to hold the loot.
- **Speed.** With the per-ship speed of `0x00612930`, the prey convoy's slowest ship
  against the pirate's fastest: `19 * prey > 20 * own` refuses. The prey may be about 5%
  faster and no more.
- **Strength.** `7 * prey > 10 * own` refuses, so a pirate attacks while it has at least
  **70%** of the target's strength. Strength is summed over a convoy as
  `sum(crew) + sum(max(crew, artillery))`, using `field_40_crew` and
  `field_120` (see [Ship Artillery](../reference/ship-artillery.md#combat-power)). For a prey
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

