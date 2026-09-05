# Crew

The sailors aboard a ship live in one word, `ship+0x40` (summed into the merchant's
fleet crew at `0x004F7E27`). What the game decides about crewing - how many a ship needs,
how many it wants, how many the tavern will hand over - comes from two byte tables and
two routines, below; **crew [morale](#crew-morale)** is a separate word that decides
whether a crew will sail at all.

## The two per-type tables

Two adjacent byte tables, indexed by the ship type (`ship+0xE & 3`):

| address | Snaikka | Crayer | Cog | Holk | meaning |
|-|-|-|-|-|-|
| `0x673660` | 5 | 8 | 10 | 12 | **minimum sailors to sail** |
| `0x673664` | 10 | 16 | 30 | 24 | **full crew** - the count the sailing math rewards up to |

The minimum is what the trade-route departure check compares the crew against (see
[Trade Routes](./trade-routes.md#crew)), and what several spawn paths write straight
into `ship+0x40` (e.g. `0x00519FE5`). It does not change with armament or upgrades.

The full-crew table is a different figure: it is the crew size the hold is laid out for.
Every sailor above it takes **2 barrels of hold**: the ship-totals recompute writes
`(crew - full) * 400` raw units (200 per barrel) into `ship+0x11C` when crew exceeds the
table (`0x00518377`; the same recompute runs on hire and dismissal at `0x005184B3` and
`0x00518571`), and free space is `capacity - ship+0x11C - cargo` wherever the game asks
(`0x00505DA1`, `0x005154F7`, `0x0051A4FB`). Mounted weapons take their space through the
same field (`0x0051A582`). Crew size has no effect on speed - see
[Speed](../ships.md#speed). The hire cap below fills toward the full-crew figure. Note the Cog
wants more sailors than the Holk in both columns' spirit: 30 full versus the Holk's 24.

## How many the ship still wants: `0x005184F0`

`thiscall(ship)`, the single authority on "how many sailors can this ship take". The
tavern's Sailors page clamps its input field with exactly this routine (two calls at
`0x005D517C`/`0x005D518F`, overwriting an entry that exceeds it), and the
[hire operation](../operations/0004-hire-sailors.md) re-clamps every request through it.

```
wanted = cargo_term / 400  +  max(0, full_crew[type] - crew)
room   = max(20, [ship+0xF] * static[type] + capacity / 1000)  -  crew
return   min(wanted, room)          (0 if the cargo recompute went negative)
```

- `cargo_term` is the figure `0x005182B0` recomputes into `ship+0x118` by walking the
  24 ware amounts at `+0x54` with the barrels/loads scale table `0x00672C14` - cargo,
  not weapons; its exact meaning is not pinned.
- `static[type]` is the first dword of the per-type record at `0x0066E030 + type*0x18`
  (3/5/8/10). The byte at `ship+0xF` is unidentified (plausibly the build grade a
  shipyard's experience sets - better yards produce ships with more capacity).
- `capacity` is the raw dword at `ship+0x10`.

## Who the tavern lists

The Sailors page builds its ship list with the ship collector `0x00504AC0` (called at
`0x005D4CA6` with mode 0), whose case walks the merchant's ship chain and keeps a ship
only if **all three** hold (`0x00504B59`..`0x00504B6F`):

- the ship's town (`+0x39`) is the tavern's town;
- the **status** (`ship+0x134`) is **`0` or `1`** - lying in port or holding with a
  pending departure, the two stationary states; a ship entering the port (`3`) or
  finishing a stop (`2`) is not offered;
- the **trade route is not active** (`ship+0x136` bit `0` clear).

A different tavern list is looser: the ship picker the captain and pirate hires use
(`0x005D7DE0`, callers `0x005CF623`/`0x005D7043`) takes the whole in-town family,
`status < 4` at `0x005D7E68` - which is the filter an earlier version of this page
wrongly attributed to the Sailors page. How many sailors are on offer is the separate,
merchant-side computation `0x004F6CA0` - see
[Sailor Pools](../merchants/sailors.md).

Status values, as far as they are pinned - each of the in-port family has its own
handler in the ships tick's jump table (`0x00507CB0`, index bytes `0x00507CDC`), and
**only `0` means lying at the quay**:

|Status|State|
|-|-|
|`0`|**lying in port** - the dock function `0x00519C90` writes it, with the moored flag `0x20` at `+0x3C`, the arrival clock at `+0x44`, `town+0x996 += 1`|
|`1`|in port with a **departure pending** - a countdown at `+0x138`, current town copied to `+0x37`, destination to `+0x38` (`0x0050729C`, `0x00509C90`); also re-entered while the port is frozen or blockaded (`0x0050699D`)|
|`2`|written at the end of a route stop's ware transfer (`0x00502089`, moored flag re-set at `0x0050209B`); its handler runs a `+0x138` timer and hands over to `1`|
|`3`|**entering the port** (`0x004E13FA`); its handler counts `+0x138` to `0x40`, then the dock function makes it `0`|
|`0xF`|at sea|

A ship the player sees in a town but not at the quay reads `1`, `2` or `3`. The moored
flag `+0x3C & 0x20` is no substitute for the status - `0`, `1` and `2` all carry it.

## Hiring and dismissing

Both go through operations, documented on their own pages:
[hire sailors (0x04)](../operations/0004-hire-sailors.md) and
[dismiss sailors (0x05)](../operations/0005-dismiss-sailors.md). The short version:
hiring clamps to `0x005184F0` and to the tavern's availability, then draws down the
merchant's sailor pool; dismissal returns the sailors to the town as beggars and
citizens, and a partial dismissal costs crew [morale](#crew-morale).

## Crew Morale

**Crew morale is the signed 16-bit word at `ship+0x3E`**, `0`..`0x700` (1792). Almost
everything that reads it reads the **high byte** `ship+0x3F` - morale `/ 256`, an
effective level `0`..`7` - so morale matters in whole 256-point steps.

Morale **drains at sea and recovers in port**:

|When|Site|Effect|
|-|-|-|
|in port, each tick after the first day docked|`0x005067F7`|`+1`, capped at `0x700`|
|at sea, each tick|`0x00506C70`|`-1`; each time the low byte wraps to `0xFF` (a 256-boundary crossed downwards) the flag `ship+0x3D` bit `0x2` is set, and a high byte that has fallen below `-1` is pulled back to `0xFF` - a spent crew's morale keeps counting down through `-1..-256` at sea|
|a **partial** crew dismissal (`count < crew`)|`0x00537EAF`|`morale -= count * 2560 / (crew + 1)`, floored at 0|
|**dismissing the whole crew** (`count >= crew`)|`0x00537EBA`|crew **and** morale zeroed|
|hiring onto an **empty** ship (crew was `0`)|`0x00537CD7` / `0x00537CE9`|high byte set to **4** with a captain aboard, **3** without - morale snaps to 1024 / 768|
|hiring onto a ship that **still has crew**|`0x00537CFB`|morale untouched|
|a route ship arriving at a stop|`0x00518A1D`|high byte floored to `1` so it can depart again|

### What morale does

- **A crew below morale 256 refuses to sail.** The departure check `can_sail`
  (`0x00519BA0`) reads the high byte and bails when it is `<= 0` (`test al,al; jle` at
  `0x00519BB8`). A crew worn down at sea sits below that line, at a zero or negative word,
  until the in-port `+1` has carried it back past 256.
- **The rest is forced and scaled.** On a [route](./trade-routes.md) stop, a ship whose
  high byte is `< 2` (morale `< 512`) has its dwell timer `ship+0x138` stretched by
  `0x500 - morale` (`0x005189FF`); at morale `>= 512` it leaves on the normal 64-tick
  dwell. A tired crew is made to wait - up to ~1280 extra ticks - before the route lets
  it move on.
- **It feeds [boarding combat](./sea-battles.md#boarding-and-melee).** The melee reads the
  same high byte (clamped `0`..`4`) as a strength term, so a rested crew boards harder
  than a spent one.

### The dismiss-and-rehire loophole

Because only the *crew-was-zero* branch of the hire handler resets morale, and dismissing
the whole crew zeroes it, **dismissing a ship's entire crew and then hiring a fresh crew
from zero restores morale to full (768, or 1024 with a captain) for the price of the new
sailors**. Topping up a tired crew - hiring onto a ship that still has some aboard - never
resets it, so the low morale persists. A worn-out ship recovers instantly by being emptied
and re-crewed, rather than by resting in port.
