# Tavern Interaction
The tavern interaction operations are enqueued by the tavern's panel when switching between the tavern's pages.
The following fields have been identified:

```c
struct operation_tavern_interaction
{
  int field_0_opcode;
  int field_4_rand;
  int field_8_merchant_index;
  int field_C_town_index;
  tavern_interaction field_10_interaction_type;
};
```

The handler is `0x0053C2C0`. It bounds the town index against the town count
(`0x006DE4B0`), then dispatches `interaction_type - 1` through the ten-entry jump table
at `0x0053C810`:

|Type|Handler|
|-|-|
|1|`0x0053C3FA`|
|2|none|
|3|`0x0053C531`|
|4|none|
|5|`0x0053C434`|
|6|`0x0053C2FB`|
|7|`0x0053C472`|
|8|`0x0053C531`|
|9|`0x0053C7A3`|
|10|`0x0053C619`|

Several of the handlers lock a tavern person to the interacting merchant: they store the
merchant index into a byte only while it still reads `0xFF` (nobody), which is what makes
the person unavailable to everyone else. Type 6 does this with `town + 0x83D`.

The panel sends a page's own type with the real merchant index when the page is opened and
the same type with an **invalid** merchant index when it is left, which is how a lock is
taken and released; type 10 is sent when the window closes. Observed types: `9` for the
side room, `4` for the sailors page, and `255` on entering the tavern, which is past the
table and does nothing. `field_4_rand` is not always a random number - type 9 uses it as a
task index with a variable slot in its upper half.

Depending on the interaction type, one of the following actions may be done.

## 1
Locks the person behind `town + 0x83E` (`0x0053C3FA`). No investigation - a
non-criminal acquaintance.

## 3 and 8
The tavern's hireable captains and pirates (`0x0053C531`): these people are
**auto-trader records** (the array at `[0x006DD7A0]`, stride `0x10`, count
`[0x006DD892]`), so the lock is the record's own merchant byte at `+0xF` rather than a
town byte, and `field_4`'s low word carries the record index (its high word feeds the
investigation roll). The criminal roll below runs **only for interaction type 8** - the
pirate; type 3, the captain, locks the same way but is legal to talk to.

## 5
Locks the person behind `town + 0x83F` (`0x0053C434`). No investigation.

## Weapons Dealer (type 6)
Handler `0x0053C2FB`, lock `town + 0x83D`. If the interaction's merchant index is
invalid, the weapons dealer is unlocked and nothing else happens (a merchant navigated
away from the page).

Otherwise, if the weapons dealer was unlocked, he is locked to the merchant and a
criminal investigation might be started. An investigation starts only if ALL hold:
- the merchant is not the alderman (`[0x006DE52D]`);
- the merchant is not the mayor of this town (`town + 0x6F1`);
- the town is not sieged, blockaded, boycotted or under pirate attack
  (`flags & 0xE10 == 0`);
- `(rand & 0x3FF) < 102` - a **~10% roll**;
- `[town + 0x9EC] < now + 0x200`.

The last condition is vestigial: the timestamp is only ever written to `now` (on a
successful roll) and zeroed at town init, so it can never be `>= now + 0x200` and the
check never blocks. **There is no cooldown**: every page entry (the lock transitioning
free -> taken) is an independent roll - switching pages and back, or closing and
reopening the tavern, rolls again.

On success a [criminal investigation](../scheduled-tasks/0005-criminal-investigation.md)
task is scheduled to `(now + 0x200) | 0x80` (two days out, afternoon) with **crime
type 0** ("criminal plans"), and the timestamp is set to `now`.

## Burglar (type 7)
Handler `0x0053C472`, lock `town + 0x83C`. Handled like the weapons dealer - the same
10% roll, the same crime type 0, its own vestigial timestamp at `town + 0x9F4` - except
the exemptions for alderman, local mayor and town status don't exist. The pirate
(type 8 above) works the same way, with its timestamp at `town + 0x9F0`.

## 9
The side room, where the tavern's [mission offers](../letters/71-tavern-missions.md) are
taken. The handler (`0x0053C7A3`) reads a task index from `+0x4` and a variable slot from
`+0x6`, requires the task to carry opcode `0x1B` and the slot to be below the task's
`+0x12`, and then

- with an **invalid** merchant index writes `0xFFFFFFFF` into that variable
  (`0x0053C7F0`), releasing the offer,
- with a **valid** one writes the merchant index into it, but only while the variable is
  still free (`0x0053C808`), locking the offer to him.

A locked offer is skipped by every other merchant's side room, so the lock is what stops
two players taking one mission.


## Leave
Type 10 (`0x0053C619`) releases every lock the interacting merchant holds in that town,
each guarded by "only if it is mine":

- the four bytes `town + 0x83C` .. `town + 0x83F` go back to `0xFF`,
- the town's auto-trader chain (`town + 0x82E`) is walked and a record whose merchant
  (`+0xF`) is this merchant is released - the tavern's captains and pirates,
- the merchant's tavern mission offers in that town are walked with `0x004D7900` and any
  whose lock variable holds this merchant is set back to `0xFFFF` (`0x0053C772`) - see
  [Tavern Missions](../letters/71-tavern-missions.md). The slot is bounded against the
  task's `+0x12`.

That last search never finds anything in practice: it compares the letter index it found
against the merchant count at `0x006DE4AA` instead of the letter pool size at
`0x006DD736`, at `0x0053C6F4` and again at `0x0053C78F`, so it gives up on any letter past
the first few dozen pool slots. Closing a tavern window therefore leaves side room offers
locked - see [Tavern Mission Lock Leak](../bugs/tavern-mission-lock-leak.md).

Every release is guarded by a comparison against the operation's merchant index
(`op+0x8`), so it only releases locks held by that merchant - an operation carrying an
invalid merchant index releases nothing here.

The panel builds this operation in three places, all in the tavern panel object
(`[0x006E54F8]`): `0x005A6604` sends type 10 with merchant `0xFFFFFFFF`, `0x005A7A7D`
sends the type from a register with the merchant count (also an invalid index), and
`0x005A7BA0` sends the type in `panel+0xB16` with the real player merchant.

Observed in game: switching from the side room to another tavern page releases the mission
lock, while closing the tavern window with a right click leaves it held - see
[Tavern Mission Lock Leak](../bugs/tavern-mission-lock-leak.md).
