# Sea Battles

A sea battle is a real object with a vtable, stepped by the engine over game ticks. The
same object and the same step loop serve a battle the player watches on the local map and
one the player declines and leaves to resolve itself - the difference is only who steers.

A pirate engaging prey reaches this subsystem through `0x0050BC40` (see
[The Pirate AI](../pirates/ai.md)), which sets convoy status `0x14` and hands both parties
over.

## Two Battle Classes

Two vtables exist, and both their objects live in the pool at `0x006E55D0`:

|vtable|slot 1|slot 5 (step)|slot 7 (report)|allocated by|initialised by|
|-|-|-|-|-|-|
|`0x0067AD44`|`0x00611740`|`0x00606CC0`|`0x0060D299`|pool methods `0x00611AE0`, `0x00611CD0`|`0x00603A70`, `0x00603BC0`|
|`0x0067AAA0`|`0x00602990` (deleting destructor, body `0x005FB9F0`)|`0x005FD510`|`0x005FE7E0`|`0x00611D80`|`0x005FBFE0`|

**A ship-against-ship battle uses the `0x0067AD44` class.** `0x00611CD0` is the pool's
find-or-make and returns a slot index; it is called from the scrollmap engagement code
`0x0050B250`, which also sends the witness operation `0x9F`. Hooking all three call sites of
`0x005FBFE0` caught nothing across repeated pirate attacks - including attacks the player
entered and fought by hand - while `0x00611CD0` fired on every one, so `0x005FBFE0` belongs
to a different path.

The heavyweight class's destructor body `0x005FB9F0` is where the aftermath lives: the piracy
charge roll (`0x005FBC8B`) and all eight calls to the per-convoy disposal `0x0050BFD0`, which
frees ships, prizes and crew.

## Sea Battle Struct

Offsets confirmed on the `0x0067AD44` class. Fields marked `[side]` are two-element arrays
indexed by side, with the stride of their type.

|Offset|Meaning|
|-|-|
|`+0x00`|vtable|
|`+0x04`|array of pointers to [ship-in-battle](#ship-in-battle-struct) objects, indexed by battle-ship index|
|`+0x0C`|word array chaining a side's ships; `0xFFFF` ends the chain|
|`+0x10`|side 0 chain head (word)|
|`+0x14`|number of battle-ship slots; chain indices are valid while below it (`0x0060779B`)|
|`+0x5C`|non-zero while the player is in the battle on the local map (`0x006074C0`, `0x00607878`)|
|`+0x15C`|battle countdown word; `0xFFFE` is written at `0x0060788F` when the battle is over|
|`+0x35A`, `+0x35C`|the two sides' merchant indices|
|`+0x559[side]`|per-side target lock; set to `0xFF` when the locked target leaves the fight|
|`+0x65C`|side 1 chain head (word)|
|`+0x65E`, `+0x660`|the two sides' convoy indices|
|`+0x662`|the battle index operation `0x96` attaches the player to|
|`+0x668`|effects/projectiles array, count in `+0xACC`|
|`+0x670`|the byte handed to every per-ship update call|
|`+0xA8C`|step counter, `0`..`24`; reset when `n * 40 >= 1000` (`0x00607496`)|
|`+0xA90[side]`|summed gunnery strength, `0x0062125B` per ship (dword)|
|`+0xA98[side]`|highest crew on the side (word)|
|`+0xA9C[side]`|highest boarding power: `crew < cutlasses ? crew * 2 : crew + cutlasses` (word)|
|`+0xAA0[side]`|highest `ship+0x120` on the side (dword)|
|`+0xAA8[side]`|highest `0x00612930(ship_id)` on the side (dword)|
|`+0xAB0[side]`|per-side scale factor used by the ship AI; written `1` at `0x005FC5EC`, `0x006053E2`, `0x0060574D` (byte)|
|`+0xAB4[side]`|mean battle-map X of the side's live ships (dword)|
|`+0xABC[side]`|mean battle-map Y of the side's live ships (dword)|
|`+0xAC4`|pending outcome kind: `0xFF` initially (`0x00603B2A`), `0` for sunk or taken (`0x006077C5`), `2` for plundered (`0x0060C3C8`)|
|`+0xAC6`|the battle-ship index `+0xAC4` refers to (word)|
|`+0xAC9`|set to `1` by the ship AI the first time a ship on the side breaks off|
|`+0xACC`|effects count (word)|
|`+0xAD2`|battle type|
|`+0xADC`|[PRNG state](#prng)|
|`+0x1E94`|packed `(crime_type << 10) \| chance_permille` - see [Criminal Investigation](../scheduled-tasks/0005-criminal-investigation.md)|
|`+0x1E98`|the witness town|
|`+0x2F20`|bystander count|

## Ship-in-Battle Struct

One object per participating ship, constructed at `0x0061A4E0`. Two vtables share the shape,
`0x0067AAD0` and `0x0067B0D4`; slot `0` of both is the per-ship update the step loop calls
(`0x00621770` and `0x0061A9E0` respectively - two compilations of the same routine).

|Offset|Meaning|
|-|-|
|`+0x00`|vtable (`0x0061A521`)|
|`+0x04`|world ship id (dword), indexing the ships array with stride `0x180`|
|`+0x0A`, `+0x0E`|**battle-map X and Y** (signed words); zeroed in the constructor|
|`+0x11`|heading, `0`..`31`; `(h + 0x10) & 0x1F` is the reciprocal (`0x00621791`)|
|`+0x26`|sail/speed setting, `4` on construction|
|`+0x128`|target or boarding partner (word), `0xFFFF` for none|
|`+0x12A`|state flags, see below; `0` on construction|
|`+0x12C`|back-pointer to the battle|
|`+0x130`, `+0x131`|gun reload timers, reset to `0x96` = 150 (`0x0061E466`, `0x0061E57A`)|
|`+0x13D`|side index, from a constructor argument; indexes `battle+0x559`|
|`+0x142`|the side the ship **started** on; `+0x13D != +0x142` is how the report detects a capture (`0x0060D7B5`)|
|`+0x140`|**crew when the battle began** (`ship+0x40`, `0x0061A58C`)|
|`+0x144`|**hull when the battle began** (`ship+0x18`, `0x0061A5A2`)|
|`+0x148`, `+0x149`|applied sail setting (`4` initially) and its `0x4B` change cooldown|
|`+0x14A`|boarding role: `0` boarder, `1` boarded (`0x00609D1E`, `0x00609D2B`)|

The battle-map position lives here and **only** here. A ship's world-chart coordinates
`ship+0x1E`/`+0x22` hold the position where the engagement began and do not move for the
battle's duration, so they are the wrong field to read for anything happening inside the
fight.

### State flags at `+0x12A`

|Bit|Meaning|Set by|
|-|-|-|
|`0x01`|the ship is breaking off / fleeing|the per-ship AI; operation `0x9B`|
|`0x02`|sunk, out of the fight|`0x006219F0`, and the step loop at `0x006077BD`/`0x00607855`|
|`0x04`|captured|the takeover routine, `0x0060CA7F`ff|
|`0x08`|grappled, boarding in progress|`0x00609CDA` and `0x00609CEE`, on both ships at once|
|`0x10`|firing / aiming|set `0x00623EEA`, cleared `0x00624403`|
|`0x20`|disengaged, gone from the fight|operation `0x9A`|

`0x26` (`0x02 | 0x04 | 0x20`) is the "no longer part of the fight" mask: such ships are
skipped when the side aggregates are recomputed (`0x006072EE`, `0x0060794D`). A ship with
`0x20` is also dropped as a gunnery target (`0x0061E498`, `0x0061E5AC`, `0x0061F27F`,
`0x0061F2C9`, `0x006034D0`) and drawn with a marker (`0x0056C3CF`, `0x0056CACB`).

The UI's per-ship status bits are assembled at `0x00610F40`.

## The Step Loop

Vtable slot 5, `0x00606CC0`, thiscall. Each step:

1. **Recompute the side aggregates.** It walks both chains - side 0 from `+0x10`, side 1 from
   `+0x65C`, through the word array at `+0xC` - and rebuilds `+0xA90` through `+0xABC` for each
   side, skipping ships flagged `0x26` (`0x00607258`..`0x00607484`). `+0xAB4`/`+0xABC` are means:
   the sum of the side's ships' `+0xA`/`+0xE` divided by the count. **With a count of zero the
   code writes the constants `0x1100` and `0xF02`** (`0x0060746E`, `0x00607479`); the
   heavyweight class writes `0x880` and `0x781` instead.
2. **Update every ship**, on both sides, through `call [obj_vtable + 0]` with `battle+0x670` as
   the argument (`0x006075D9` for side 0, `0x006076B9` for side 1).
3. **Advance the effects array** at `+0x668`, `+0xACC` entries, through `0x00603130`.
4. **Test whether the battle is over** (below).
5. **Advance the step counter** `+0xA8C`.

The local map runs at 3375 ms per tick - see [Time](../time.md).

### Ending the Battle

`0x006076FB`..`0x00607878` works from booleans gathered during the per-ship walk, one pair per
side:

|Booleans|Meaning|
|-|-|
|`[esp+0x16]`, `[esp+0x17]`|the side still has a ship without bit `0x20`|
|`[esp+0x12]`, `[esp+0x13]`|the side has a ship with free hold `>= 0x7D0`: `ship+0x10 - ship+0x11C - ship+0x118 >= 2000`|
|`[esp+0x14]`, `[esp+0x15]`|the side has a ship carrying goods|

When a side has no ship left in the fight, the winner takes ships one at a time: `0x02` is ORed
into the loser's flags, `+0xAC4` is set to `0` with `+0xAC6` naming the ship, and `+0x15C`
becomes `0xFFFE`.

## Auto-Resolved Battles

Declining the prompt does not hand the fight to a formula or a table. **The battle runs the
step loop above, headlessly, over game ticks, with the AI steering both sides.**

The state that proves it is the state a simulation has to carry. Sampling the battle object
every tick through a whole battle - once with the player steering, once declined and left
alone - gives the same picture both times:

- Six to nineteen dwords of the object change on **every** step, and they are the same fields
  in both cases: the step counter `+0xA8C`, both sides' mean positions `+0xAB4`/`+0xAB8` and
  `+0xABC`/`+0xAC0`, and the PRNG `+0xADC`.
- `+0xADC` takes an unrelated value every step, so the fight draws on its RNG continuously
  rather than once.
- `+0xAB4`/`+0xAB8` climb smoothly while the player holds a heading and **wander up and down**
  under AI control - the AI's course is the less straight of the two.
- Damage arrives in **many separate events on both ships** - nine of them across 22 ticks in
  one declined battle, on top of six separate writes by the crew casualty routine
  `0x00620177` - then stops, while the mean positions run down toward zero for a further forty
  ticks and the battle ends the tick after they arrive.
- Battle length varies with what happens: 26 steps for one fight settled by hand against 74
  for a declined one.

Both sides' mean positions falling together is the shape of a chase: the loser runs for the
edge of the map and the winner follows, and the fight ends when they get there.

The plunder and takeover routine `0x0060C1C9` is shared by both paths - the declined battle
reaches it, and so does the interactive class through its per-ship AI
(`0x00621770` -> `0x00621CEE` -> `0x0060C1C9`). It is the routine that raises the crime type on
a capture and pays the prize money; see
[Criminal Investigation](../scheduled-tasks/0005-criminal-investigation.md).

## Entering and Declining: Operation `0x96`

The prompt's two answers are one operation. Handler `0x0060FEC9` compares the index it is given
against the player merchant `[0x006DFC14]` and toggles **`[0x006E59CC]`**, the battle the player
is attached to: `0x0060FF1E` sets it from `battle+0x662`, `0x0060FF0F` clears it to `0xFF` for
none.

## Boarding and Melee

When two ships grapple - flag `0x08` on their `+0x12A` - the per-ship AI step `0x0061F1BF`
resolves the fight by **attrition**: every tick each ship inflicts casualties on the other
in proportion to its own **strength** against the other's, and when one side's crew is
emptied the winner captures it (its crew is set to `1` and the takeover routine
`0x0060C1C9` runs). There is no single "who wins" comparison - a lopsided fight is over
fast, a close one is bloody on both sides, and the per-tick rolls add noise.

Each ship's strength (`0x0061F347`, mirrored for its partner at `0x0061F447`):

```
crew_eff       = crew + (1 if a captain is aboard)          ; crew = ship+0x40
boarding_power = crew_eff + min(cutlasses, crew_eff)        ; cutlasses = ship+0x154
base           = ((morale_level + 18) * boarding_power + 10) / 20
strength       = base * (100 + captain_combat * 6 / 17) / 100   ; only with a captain aboard
```

The four things that decide it:

- **Crew count** is the backbone - `boarding_power` is linear in it. Two unarmed,
  captainless ships reduce to `strength = 18 * crew / 20`, so there the **larger crew
  wins**, nearly deterministically. That is the common trader-vs-trader case.
- **Cutlasses** (`ship+0x154`): each one lets a sailor count **double** in
  `boarding_power`, capped at one per sailor. A ship carrying at least as many cutlasses
  as crew fights at twice its crew; with none, at face value.
- **The captain's combat skill** (the captain record's combat byte `+0xB`, raw `0`..`255`;
  the tavern offer panel shows `raw / 50`) is a **multiplier** - `1 + 0.00353 * raw`, so up
  to about **1.9x at 255**, roughly **+18% per displayed level** (`combat * 6 / 17` at
  `0x0061F41C`..`0x0061F428`, the `0x78787879` magic being the divide by 17). It applies only when a captain is aboard - which is also where the
  `+1` crew and the morale bonus come from.
- **Crew [morale](./crew.md#crew-morale)** enters as `morale_level + 18`, where
  `morale_level` is the morale high byte (`ship+0x3F`) clamped to `0`..`4`. A rested crew
  (level 4) boards at `(4 + 18)` against a spent crew's `(0 + 18)` - about **+22%** at full
  morale.

For a **human**-owned ship the base term also takes a difficulty swing: `+2` when the
combat-difficulty stepper `[0x006CC3E8]+0x12` is `0` (the two easiest presets), `-2` when it
is `2` (the hardest), nothing in between (`0x0061F3B8`).

So a much smaller crew can still win: 20 sailors with 20 cutlasses under a skilled captain
outfight 30 bare crew comfortably. "Most sailors wins" is only the special case of two
unarmed, unofficered ships.

## The Battle Report Letter

The "Naval battle" letter is built by vtable slot 7, `0x0060D299`, and arrives **only for
battles that were left to resolve themselves**.

Its opening sentence comes from the eight-entry pointer table at `0x006A2240`, read at
`0x0060DB70` and neighbours:

|Index|Sentence|
|-|-|
|0|You have put the attackers to flight!|
|1|You have defeated the enemy!|
|2|You were defeated by the attacker, but managed to escape!|
|3|You lost the sea battle!|
|4|Your attack was successful!|
|5|Your enemy escaped!|
|6|Your attack was a failure! Your ships were defeated and had to flee!|
|7|Your attack was a total failure! You lost the sea battle!|

The sentence is **chosen from tallies, not rolled**: three byte counters accumulated during the
ship walk are compared as `2 * (B - A) <= C` at `0x0060DB3A`, so the letter describes an
outcome the ship states already hold.

Three more pointers follow the sentences - the section headings `Own ships:`,
`Captured ships:` and `Enemy ships:` at `0x006A2260`..`0x006A2268`. The per-ship line then has
**five** formats, and only four of them are in this table: the fifth sits in the neighbouring
"Seizure" string table and is reached through `0x006AA180`.

|Format|Pointer|Chosen at|Witness weight|
|-|-|-|-|
|`%s %s: %d%%`|`0x006A226C`|`0x0060D832`, `0x0060D8FE`|`0x32` = 5%|
|`%s %s: sunk`|`0x006A2270`|`0x0060D771`|`0x64` = 10%|
|`%s %s: %d%%, captured`|`0x006A2274`|`0x0060D7E1`|`0xC8` = 20%|
|`%s %s: %d%%, fled`|`0x006A2278`|`0x0060D88B`|`0x32` = 5%|
|`%s %s: %d%%, plundered`|`0x006AA180` -> `0x006A9A9C`|`0x0060D8BC`|`0x32` = 5%|

The percentage is the ordinary `health * 100 / max_health` (`0x0060D7CE`..`0x0060D7D2`), and the
weight each branch loads into `edi` is the per-mille contribution that fate makes to the piracy
charge - see
[Criminal Investigation](../scheduled-tasks/0005-criminal-investigation.md).

**Capture is detected by comparing two side bytes** on the ship-in-battle object: `+0x13D`, the
side the ship is on now, against `+0x142`, the side it started on (`0x0060D7A0`..`0x0060D7B7`).
When they differ the ship is printed **twice** - once with `, captured` and once with the plain
line - which is why a taken ship appears under both `Captured ships:` and its owner's section.
The `, fled` and `, plundered` branches are selected by bits `0x02` and `0x40` of a per-ship fate
byte tested at `0x0060D862` and `0x0060D893`; that byte is not the `+0x12A` state flags and its
origin has not been traced.

The sentence table continues past the per-ship formats, at `0x006A227C`, with the sentences for
an attack on a **town** - "You have protected the town of %s...", "...relieved the city's bank of
%d..." - so town assaults report through the same machinery.

## PRNG
Every battle has its own [PRNG](https://en.wikipedia.org/wiki/Pseudorandom_number_generator) state at offset `0xadc`:
```c
signed __int32 __thiscall get_battle_rand(sea_battle *this)
{
  signed __int32 v1; // edx
  unsigned __int32 v2; // eax
  signed __int32 result; // eax

  v1 = 1153374643 * this->field_ADC_prng_state;
  v2 = -1576685469 - v1;
  this->field_ADC_prng_state = -1576685469 - v1;
  if ( ((99 - (_BYTE)v1) & 1) != 0 )
    result = (v2 >> 1) | 0x80000000;            // shift in a leftmost 1
  else
    result = v2 >> 1;                           // shift in a leftmost 0
  this->field_ADC_prng_state = result;
  return result;
}
```

The state advances on every step of the loop above, in a battle the player steers and in one
left to resolve itself alike.

The same generator is **inlined** wherever the compiler could: the multiplier
`0x44BF19B3` (= 1153374643) and the constant `0xA205B063` (= -1576685469 unsigned) appear
verbatim in the piracy-charge roll, twice - once per side, at `0x00603E37`/`0x00603E8D` and
`0x00603ECE`. Each copy draws once, compares `rand % 1000` against the chance in the low ten
bits of `+0x1E94`, and on success draws again and reports the crime type `+0x1E94 >> 10`
together with that side's convoy (`+0x65E`/`+0x660`) and merchant (`+0x35A`/`+0x35C`) - see
[Criminal Investigation](../scheduled-tasks/0005-criminal-investigation.md). An inlined copy is
easy to mistake for a second, private generator; there is only one.
