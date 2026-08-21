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

## 3 and 8

## 5

## Weapons Dealer
If the interaction's merchant index is invalid, the town's weapons dealer is unlocked, and no other action is performed.
This happens if a merchant navigates from the weapons dealer page to a different page.

Otherwise if the town's weapons dealder is unlocked, it'll be locked to the merchant, and a criminal investigation might be started.
An investigation is started only if all of the following conditions are met:
- The merchant is not the alderman
- The merchant is not the mayor in the particular town
- The town is not sieged, blocked, boycotted or under pirate attack
- The following formula is true: `(rand & 0x3ff) < 102`
- The following formula is true: `weaponsdealer_timestamp < now + 0x200`

If all conditions are met, a criminal investigation scheduled task is scheduled to `(now + 0x200) | 0x80`, and the weapons dealer timestamp is set to `now`.

## Burglar
The burglar is handled like the weapons dealer, except the exceptions for alderman, local mayor and town status don't exist.

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
