# Pirates
Pirates are not scripted. The `.p2m` interpreter that runs letter and mission scripts
(`0x004ECF64`) has exactly one caller, the letters task, so no bytecode ever touches a
ship. Everything below is native code: a state machine inside the per-tick ships update,
plus a handful of scheduled tasks that maintain the pirate population.

There are two different things called "pirate":

- a **pirate ship** roaming the map, owned by nobody (`field_0_merchant_index` = `0xFF`)
  and flying status `0x12`. These belong to *bands* based at hideouts, and are what this
  chapter is mostly about. Both fields are written together at `0x005151A4`, where a ship
  puts to sea as a raider: status `0x12`, `field_15C_is_pirate`, merchant index `0xFF`, and
  a convoy record of its own from `0x005062F0`. `0x00516225` does the same alongside a full
  refit.
- a **pirate captain**, one of the tavern characters an
  [auto trader record](./auto-traders.md) can be, whom the player can put in command of
  one of his own ships. Handing the ship over (operation `0x0D`) only sets
  `field_15C_is_pirate`, and only on a ship already at sea under its normal status `0x0F`;
  it leaves the merchant index alone. But a raider actually sailing for its pirate captain
  reads `field_0_merchant_index` = `0xFF`, like any other pirate ship - measured in a live
  save, where two ships just given to pirates read `0xFF` while their sister ships under
  ordinary captains still read the player's index. What keeps such a ship tied to its owner
  is `field_15D`, not the merchant index.

The two halves of the subject have pages of their own:

- [Bands and Hideouts](./pirates/bands.md) - where raiders come from, where they go to be
  repaired, and what a homecoming pays out.
- [The Pirate AI](./pirates/ai.md) - how a raider at sea picks a target, decides whether to
  attack it, and paces its raids.

## Ship Fields
|Field|Meaning|
|-|-|
|`field_15C_is_pirate`|set by operation `0x0D` (`0x005386C0`) on a ship that is at sea; also ORs `0x18` into `field_3D`. This is the flag a hired pirate's ship carries|
|`field_15D`|the merchant a pirate belongs to, `0xFF` for a free pirate. Written by the ship spawner `0x00509250` from the ship's own merchant index, and cleared again if the ship reaches a hideout under half hull|
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
