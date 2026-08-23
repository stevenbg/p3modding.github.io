# Auto Traders
Captains, office administrators and pirate captains are all the same 16-byte record. The
array lives behind the ships container at `0x006DD7A0` (array pointer at `+0x0`, stride
`0x10`, allocated capacity in the word at `+0xF2` = `0x006DD892`):

```c
struct auto_trader
{
  unsigned __int16 field_0_next_auto_trader_index;
  signed __int16 field_2;
  int field_4_timestamp;
  unsigned __int8 field_8;
  unsigned __int8 field_9_navigation_skill;
  unsigned __int8 field_A_trade_skill;
  unsigned __int8 field_B_combat_skill;
  __int16 field_C_daily_wage;
  char field_E;
  unsigned __int8 field_F_merchant_index;
};
```

Every byte is accounted for, and none of them is the record's own index - a record does
not know where in the array it sits, which matters because
[the skill ceilings are a property of the slot](./auto-traders/skill.md#the-ceiling-belongs-to-the-slot).
Note that `field_2` is written and read as **two bytes**, not one word: `+0x2` is the first
name id and `+0x3` the surname id.

The rest of this chapter covers what happens to a record over its life:

- [Skill](./auto-traders/skill.md) - the one writer of the three skill bytes, the ceilings,
  and how a captain's skills end up where they do.
- [Wages](#wages) - below, since the wage is recomputed by whatever touched the record.
- [Administrators](./auto-traders/administrators.md) - the one kind of auto trader with a
  growth path of its own.
- [Retirement](./auto-traders/retirement.md) - how a record's life ends.

## The Two Kinds
`field_8` discriminates them (the check is `0x004FE150`, true for `field_8 > 0x20`). The
record initializer (`0x004FDF50`) fills `field_8` with a random byte and reduces it `% 11`
for captains, giving `0`..`10`, so the range encodes the kind: captains always pass
`<= 0x20`, and the pirate captains - one record per town, maintained by the
[spawn task](#captain-and-pirate-spawning) - fail it. The pirates are the tavern
characters a ship can be handed to, identified by matching the records' name ids against
the pirate captains seen in-game.

For a pirate `field_8` doubles as the **greed byte**: the loot share a tavern pirate
demands is `25 + 5 * ceil(field_8 / 32)`, i.e. 35%..65% - verified against seven live
pirates.

The initializer also rolls those two [name ids](./ui/name-banks.md) (modulo the
name-registry counts `0x006DDB70` and `0x006DDB74`), splits a 600-point budget randomly
across the three skills, and stamps `field_4` from the current date serial - see
[Retirement](./auto-traders/retirement.md) for what that stamp is later read as.

## Where a Record Can Be
A record carries no field saying what it currently is. Its situation is expressed entirely
by **who points at it**, which is why the same struct serves three roles:

|Pointed at by|The record is|
|-|-|
|a town's `field_82E_auto_trader_chain_head` chain|sitting in that town's tavern|
|a ship's `field_42_captain_index`|commanding that ship|
|an office's `+0x2F2`|that office's [administrator](./auto-traders/administrators.md)|
|the freelist, through `field_0`|free|

The town chain and the freelist share the same link field, `field_0`. Both are walked by
following the *index* in `field_0` to the next record; the chain ends on an out-of-range
index. A town's chain **mixes both record kinds** - its hireable captains and its one
pirate captain sit on the same list, which is why each has a resolver of its own.

A town's tavern offers a captain for hire while its chain holds a captain record no
merchant employs (`field_F_merchant_index` = `0xFF`). The captain resolver
`0x005269A0(town, merchant)` walks the chain applying exactly that, preferring a captain
the asking merchant already employs, and returns the auto-trader index or `0xFFFF`; the
sibling resolver `0x005261D0` does the same for the pirate captains. Verified against a
live save: exactly the towns whose taverns showed captains had a matching chain record.

The record's merchant byte is only maintained for the player - **AI merchants' hires keep
`0xFF`** - so ship ownership comes from the ship's own `field_0_merchant_index` (AI
merchants hold low indices with two starting ships each; pirate ships and empty ship slots
hold `0xFF`).

## Array Layout and Slot Recycling
Observed live in a 128-record array: indices `0`..`47` hold one world-generation pair per
town - the founding captain of an AI merchant's starting ship (elite skills, exceeding the
600-point budget of the normal roller; verified by matching the records against the AI
ships' `field_42_captain_index` and owners) and the town's pirate. The dynamic range above
that holds spawned tavern captains and employed records. The tail is free capacity,
`memset` to `0xFF` and freelist-linked through `field_0`; the allocator at `0x005097C0`
grows the array by 64 records when it runs out, which is why references to a record are
indices rather than pointers.

`+0xF2` is the number of slots allocated, **not** a live population count - there is no
count anywhere, because free slots are a freelist rather than a tail. It is still the right
validity bound, and the one the game itself uses: the
[skill gain](./operations/0012-auto-trader-skill-gain.md) handler rejects a captain index
that is not below `[0x006DD892]` (`0x00538AC3`), exactly as it rejects a ship index not
below `[0x006DD894]`.

The **freelist head** is the word at `ships + 0xE0` = `0x006DD880`. The allocator
(`0x005097C0`) pops it: it reads the head, follows that record's `field_0` to the next free
index, stores that back as the new head, writes `0xFFFF` over the popped record's link, and
returns the popped index. When the list runs dry the head equals the capacity, which is the
grow trigger - the allocator then raises `+0xF2` by `0x40`, reallocates, and relinks the 64
new records into the freelist by writing `index + 1` into each one's `field_0`.

Records never expire, and the population circulates between taverns and decks. Because a
slot is settled when the record is **created** and nothing ever moves a record afterwards,
the only question that matters for a captain's ceilings is whether an action reuses his
record or allocates a new one:

- **Dismissing a captain does not destroy his record.** The finalizer (task `0x29`,
  `0x004DDE70`) unlinks it, writes `0xFF` into its merchant byte and links it back into the
  town's chain, so the same man - same slot, same ceilings, same skills - waits in that
  tavern to be hired again. Hiring him, dismissing him, moving him between ships: none of
  it reallocates anything.
- **An office administrator is the exception.** Operation `0x5E` frees his record to the
  freelist when he is dismissed (`0x005098B0`, which also clears the office's autotrade
  flags), and its hire path unconditionally allocates a fresh one and writes trade `= 0`
  (`0x0053DA1D`) before recomputing the wage. The record that comes back is a different man
  with a new name and new skills, who has to earn his trade skill again from nothing.
  Whatever the office appears to remember about a previous administrator is stored on the
  office, not in this array.

## Captain and Pirate Spawning
A periodic task (`0x004E2634`, rescheduling itself in [game ticks](./time.md), 256 per
day) maintains both populations. Per town it records two flags: whether the captain
resolver finds an unemployed captain - called **with the merchant count as the asking
merchant**, a value no real merchant has, so only records with
`field_F_merchant_index` = `0xFF` count - and whether the pirate resolver finds a pirate.

- **Pirates**: when fewer than 3 towns have a pirate, one is spawned into a pirate-less
  town (`0x00526A50(town, 1)` - the pirate initializer path).
- **Captains**: at 8 or more unemployed captains the task just reschedules far out
  (`+0x700` ticks = 7 days). Below that it compares the count against a demand target
  derived from fleet statistics (`0x00509930` on the ships container) and spawns a captain
  into a captain-less town when the count is at or below the target, or below 2
  (`0x00526A50(town, 0)`), rescheduling `+0x200` ticks (2 days) after a spawn and `+0x400`
  (4 days) otherwise.

No expiry logic exists in the task, and `field_4` is never compared against the current
date: an unhired captain stays until somebody hires him - including AI ships, which fill
their `field_42_captain_index` through the same resolver and unlink the captain from the
town (`0x0051A1B9`). A captain "disappearing" from a tavern is somebody else's hire, not a
timeout.

### The Dismissal Undercount
Dismissing a captain re-links him into the town's chain **still carrying the dismissing
merchant's index**; it only flips to `0xFF`, and so to generally hireable, when that town's
tavern is next opened (observed in-game). Until then the spawn task does not count him, so
the game under-counts the hireable population and spawns extras. Dismissing captains
without revisiting their taverns therefore pushes the world above the usual two hireable
captains - four were observed live.

## Wages
`field_C_daily_wage` is recomputed from the skills whenever they change, by one of two
routines. Which one runs depends on the path that touched the record, not on the kind of
record it is:

|Routine|Formula|Used by|
|-|-|-|
|`0x004FE190`|`(nav + trade + combat) / 50 + (field_8 % 11) + 10`|the [skill gain](./operations/0012-auto-trader-skill-gain.md) handler, so every captain and pirate|
|`0x004FE160`|`20 * (trade / 43) + 10`, i.e. only ever 10, 30, 50, 70, 90 or 110|the [administrator](./auto-traders/administrators.md) paths, operations `0x5E` and `0x67`|

Both check out against live saves: a captain with `39/61/125` and `field_8` = 6 reads wage
22, a pirate with `237/251/112` and `field_8` = `0xFB` reads 31, and administrators at
trade 0, 43, 86, 129 and 215 read 10, 30, 50, 70 and 110.

### What the Interface Shows Is Not the Record's Wage
For an administrator the trading office window calls `0x00500F10(office)`, which returns
the record's wage **plus** `office+0x2D2`, and falls back to `office+0x2D2 + 10` when the
post is vacant - the cost of the level 0 administrator you would get.
[`office+0x2D2`](./merchants/trading-office.md) counts the **business buildings the
merchant owns in that town**, one per building, verified across several saves and offices.
So a busy office pays its administrator a gold a day more for every business it runs, and
an office with none pays exactly what the record says.
