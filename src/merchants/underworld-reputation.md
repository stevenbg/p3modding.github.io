# Underworld Reputation

The Personal page of the trading office grades your relationship with the underworld:

- "The underworld is avoiding you."
- "Your name is being mentioned by the underworld."
- "The underworld regards you as one of their own."

The stat behind the sentence is a **signed 16-bit word at `merchant + 0x16`**. Every
change is a saturating add clamped to `[-0x8000, 0x7FFF]`. The UI reads only the high
byte (`+0x17`, i.e. `value / 256` - call that the *grade*), clamps it to ±20
(`0x005DD846`), caches it in the window (`+0xED98`), and picks the sentence at
`0x005DB8B1`:

|Grade|Sentence|
|-|-|
|< 4|avoiding you|
|4..6|your name is being mentioned|
|>= 7|one of their own|

The three strings live in the executable at `0x006A9190`/`0x006A91B0`/`0x006A91E0`
(pointer table `0x006A9700`).

## What Raises It

The deltas are data, a word table at `0x00672DD0`:

|Event|Delta|Site|
|-|-|-|
|hiring the tavern **burglar** (operation `0x4F`, `0x0053BA40`)|+256|`0x0053BB24`|
|hiring the tavern **pirate** onto a ship (operation `0x4E`)|+256|`0x0053B932`|
|a **hired pirate captures a ship** - the "your share of the goods and gold" award|+65|`0x0060D063`|
|pirate-sponsor completion paths|+65|`0x0054096F`, `0x00540CDD`|
|buying **militia weapons** from the tavern's weapons dealer ([operation `0x50`](../operations/0050-buy-militia-weapons.md))|+25|`0x0053BD00`|

One full grade point is 256 units, so each criminal hire moves the sentence ladder by
one, a sponsored capture by about a quarter, and an arms purchase by a tenth.

Operation `0x4E` is the criminal twin of operation `0x06`, the ordinary captain hire:
both unlink the person from the town's tavern chain (`0x00526AA0`, chain head
`town + 0x82E`) and write him into `ship + 0x42` - only `0x4E` charges money and
credits the underworld. The +65 award requires the capturing ship's employer byte
(`ship + 0x15D`, set only on hired pirates) to name a valid merchant; the player's own
black-flag captures do not pass through it.

## What Lowers It: Killing Pirates

A shared helper (`0x004F8D90`) subtracts **512** - two whole grade points - and every
one of its callers is about a **pirate ship being destroyed**.

The callers live in the post-battle convoy disposal `0x0050BFD0`, which the two battle
resolution sites (`0x005FBCE5`..`0x005FBDF5`, `0x00603FB0`/`0x00603FCC`) run once per
combatant convoy (`battle + 0x65E`/`+ 0x660`) and once per bystander convoy (the list
at `battle + 0x2F1C`, count `+ 0x2F20`). Its third argument is always **the opposing
side's merchant**: the bystander loop compares the convoy's owner byte against
`battle + 0x35A` and passes whichever of the battle's two merchant indices is not the
owner. All three subtraction sites require the subject to be **ownerless** - which is
what a pirate is:

|Site|Condition|
|-|-|
|`0x0050C1FB`|reached only when `convoy + 0x0 >= merchant_count` (`0x0050C0FF`), then per ship with `ship + 0x18 < 1` - destroyed|
|`0x0050C7DA`|explicit at `0x0050C7A4`: the ship's owner byte must **not** be a valid merchant|
|`0x0050C9AA`|the convoy's owner must not be a valid merchant (`0x0050C902`)|

So sinking pirates is what the underworld holds against you, at 512 per ship, and
nothing about fighting other merchants registers at all.

The helper's argument carries a second effect: it is set when the destroyed pirate ship
**had a captain** (`ship + 0x42` naming a valid captain record), and then `0x004F8DF4`
adds `merchant + 0x464` (the base reputation factor, normally 1.0) to the social
reputation float of **every town in the world** - the reward for killing a named
pirate. That is the same field a
[conviction](../scheduled-tasks/0005-criminal-investigation.md#the-reputation-cost)
subtracts from, `merchant + 0x120 + town*12`.

The same routine also maintains `merchant + 0x1B`, a counter capped at 250 that rises
by the number of pirate ships you eliminate (`0x0050C97A`) and falls, floored at 0, by
the number of ships their owner loses (`0x0050C925`). No consumer of it has been found.

**Decay**: the periodic merchant update steps the underworld word by 11 (`0x00672DE6`)
toward zero - the block at `0x004F84BD`..`0x004F84D6`, the subtraction at `0x004F84C6` -
so the reputation fades on its own. The step is unconditional and the branch is `jle`,
so a faded word never rests: 0 takes the add path and the raw value oscillates
`0 <-> 11` forever, and any magnitude below 11 overshoots (`+5 -> -6`). The UI is blind
to this - it reads the high byte - but a mod reading the raw word should expect the
bounce rather than a stable zero.

A criminal **conviction** does not touch it - that costs
[social reputation](../scheduled-tasks/0005-criminal-investigation.md#the-reputation-cost)
instead.

## What It Is Good For

Two consumers exist:

- the Personal-page sentence above;
- the **standing** word at `merchant + 0x22`, which the same periodic update drifts by
  ±1 toward `20 * (2*rank - 6 - grade)`, clamped to ±30000 (`0x004F84DC`) - rank
  raises the target, the underworld grade lowers it. Becoming mayor (operation `0x46`,
  `0x00535FD6`: requires hometown rank Patrician, writes `town + 0x6F1`) sets the
  standing to 1000 outright. The one traced gameplay gate is the town hall's
  **announce-a-celebration** action, which refuses below standing 20 (`0x005E775C`,
  see [Celebration](../scheduled-tasks/0007-celebration.md)) - so the gate works out
  to `2*rank - 6 - grade >= 1`: a Mayor-rank merchant is locked out of hosting
  celebrations from grade 2, a Patrician from grade 4.

The stat does **not** feed the tavern criminal pages (their charge roll is a flat
~10%), the bribe expectations, the verdict, the fine, or the pirates' choice of prey.
