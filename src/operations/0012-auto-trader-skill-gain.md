# Auto Trader Skill Gain
Operation `0x12` raises the skills of the [auto trader](../auto-traders.md) commanding a
ship. It is the only writer of the three skill bytes in the game.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x12`|
|0x04|u32|ship index - the record is that ship's `field_42_captain_index`|
|0x08|u32|navigation gain|
|0x0c|u32|trade and combat gain|

The handler at `0x00538A80` (operation switch case `0x00535971`) bails unless the ship
index is below the ship count and the ship carries a captain index below the auto-trader
count. For each of the three skills it adds that skill's gain to the current byte and
clamps the result against the skill's ceiling:

```python
new = (skill + gain) & 0xFF          # a byte addition
if new > cap or new < skill:         # over the ceiling, or wrapped
    skill = cap
else:
    skill = new
```

It finishes by recomputing the wage from the new skills (`0x004FE190`).

## Trade and combat share one gain field
`0x00538B16` and `0x00538B2E` are the same three instruction bytes: the handler reads
`+0x0C` for the trade gain and reads it again for the combat gain. The record's fourth
payload dword at `+0x10` is never read here and never written by either producer.

So **trade and combat always move by the same amount.** The difference between them is
fixed when the record is created and nothing but a clamp can change it: whichever of the
two is higher reaches its ceiling first and waits there while the other catches up. The
same applies in reverse - a gain aimed at trade raises combat even when combat is already
past the threshold that would have refused a combat roll, which is what lets the pair
climb well beyond that threshold. See
[the ten-day update](../scheduled-tasks/0003-ten-day-update.md#what-that-means-for-a-captains-final-skills)
for what that does to a captain's final skills.

Measured on a live save: `trade - combat` was unchanged for every captain across two dumps
259 days apart, except where one of the two had met its ceiling in between.

## The ceiling is a property of the record's slot
The ceilings come from the four-byte table at `0x00673B34` - **250, 200, 250, 150**, i.e.
displayed levels 5, 4, 5 and 3 - indexed by bits of the record's **index in the
auto-trader array**:

|Skill|Table index|
|-|-|
|navigation|`index & 3`|
|trade|`(index >> 2) & 3`|
|combat|`(index >> 4) & 3`|

So a captain's ceiling in each discipline belongs to the array slot, not to the man, and
because records are recycled through the freelist a re-hire inherits whatever the
allocator hands out.

## A skill above its ceiling is pulled down
The clamp is unconditional and runs for all three skills on every application - including
one whose gain for that skill is `0`. An over-ceiling skill is therefore cut to its
ceiling by the first operation `0x12` that reaches the record, whichever skill the gain
was meant for.

That is not a corner case. The record initializer `0x004FDF50` produces each skill by
reinterpreting the bits of a float (`0x004FE046`, `0x004FE073`, `0x004FE0AE`), giving a
roughly uniform `0..255` per skill with the sum capped at 600, so a fresh record above
150 or 200 is common. Measured on a live save: a captain with all three skills at 253 was
cut to `250 / 150 / 250` the first time the scan reached him, losing 103 raw points - two
and a half displayed levels - of trade.

## Producers
Two, and only two:

- the [ten-day world update](../scheduled-tasks/0003-ten-day-update.md), which is what
  grows captains over time;
- a pirate raider reaching its hideout, which awards a flat `50` and `50`
  (`0x00514C93`, see [Pirates](../pirates.md#bands-and-hideouts)).

Nothing else writes a skill byte, and nothing anywhere decrements one.
