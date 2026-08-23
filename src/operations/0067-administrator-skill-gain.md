# Administrator Skill Gain
Operation `0x67` raises an office administrator's trading skill by one displayed level.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x67`|
|0x04|u32|auto-trader index|
|0x08|u32|merchant index|
|0x0c|u32|town index|
|0x10|u32|the new trade skill byte|

The handler at `0x0053DDF0` (operation switch case `0x005364B6`) resolves the office
through `0x005308A0(merchant, town)`, checks that `office+0x2F2` still names the trader in
the payload, writes the byte to the record's trade skill, recomputes the wage
(`0x004FE160`) and posts a type `0x7C` event through `0x004D6530`.

Unlike a captain's [skill gain](./0012-auto-trader-skill-gain.md) there is no ceiling
table here. The only limit is in the producer, the
[ten-day world update](../scheduled-tasks/0003-ten-day-update.md), which computes
`new = (trade + 43) & 0xFF` and drops the whole thing when that wraps below the current
value. Since a freshly hired administrator starts at `0`, an administrator's trade skill
walks exactly `0, 43, 86, 129, 172, 215` and then stops - always an exact multiple of 43,
which is one displayed level. Navigation and combat are never touched.

Because there is no ceiling and nothing to lose, the outcome is not in doubt: every
administrator left in place long enough reaches level 5. Only the pace is random - the
producer rolls `(rand & 0x3FF) < 0x1B3` on each of his eligible rounds, so about 42% of
them pay.
