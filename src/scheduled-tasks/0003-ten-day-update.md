# Ten-Day Update
The task with opcode `0x03` (`0x004DDA40`) is a periodic sweep over several subsystems. The
dispatcher reschedules it at `0x004D8668` with `due += 0xA00` = 2560 [ticks](../time.md), so
it runs **once every ten days**.

This page covers the part of it that maintains the world's
[auto traders](../auto-traders.md): the call to `0x004DCEA0`, which is the only thing in the
game that grows a captain over time.

## What One Run Does
For every merchant, for every ship of that merchant:

- skip the ship if its status is `0x11`, if it carries no valid captain index, or if the
  captain's index does not belong to this run's **group** (below);
- [retire](../auto-traders/retirement.md) the captain if he is old enough;
- give the captain a skill gain, through
  [operation `0x12`](../operations/0012-auto-trader-skill-gain.md).

Then, **only for a human merchant** (`merchant+0x8` = 0), for every office of that merchant
(`merchant+0xC`, chained through `office+0x2C8`): with the same group filter and a
`(rand & 0x3FF) < 0x1B3` roll, raise the
[administrator's](../auto-traders/administrators.md) trade skill by one level through
[operation `0x67`](../operations/0067-administrator-skill-gain.md).

Nothing requires a ship to be sailing, carrying cargo or doing anything at all, and a record
sitting in a tavern is never reached - the sweep only ever walks merchants' ship chains.

## The Two Growth Paths
The owner's control word decides which.

An **AI merchant's** captain (`merchant+0x8` non-zero) gets a flat `8` in both gain fields,
applied by calling the operation switch `0x00535760` directly - no queue, and no threshold
test at all. All three of his skills rise by 8 whenever his group comes up, until each meets
its own ceiling.

A **human player's** captain (`merchant+0x8` = 0) gets one roll of `rand & 0x3FF`, taking one
of three branches:

|Roll|Gain field written|Skill whose threshold is tested|
|-|-|-|
|`0x000`..`0x155`|navigation|navigation|
|`0x156`..`0x2A9`|trade and combat|trade|
|`0x2AA`..`0x3FF`|trade and combat|combat|

The gain is `rand % 51`, so `0`..`50`. Nothing is enqueued unless the skill in the third
column is **below the record's navigation ceiling** `T`: the enqueuer reads a second copy of
the [ceiling table](../auto-traders/skill.md#the-ceiling-belongs-to-the-slot) at
`0x00672824`, indexed with the run's group, which for a ship that passed the group filter is
the record's own `index & 3`. The ship is skipped before the roll if all three skills have
already reached `T`. The two lower branches differ only in which threshold they test,
because [both write the same field](../auto-traders/skill.md#trade-and-combat-share-one-gain-field).

Which of those branches a captain's career actually ends on - and why a slot's trade and
combat ceilings are often unreachable - is
[Where a Captain Ends Up](../auto-traders/skill.md#where-a-captain-ends-up).

Each enqueue also costs one slot of the operation queue's headroom
(`0x34 - [0x006DF346]`, read once at the top of the run); once that is spent the rest of the
run is silently dropped.

## The Group, the Counter, and the Annual Halt
A run does not touch every captain. It only looks at records whose `index & 7` equals a
**group** number, so a given captain comes up about every eighth run - roughly every 80 days,
four times a year.

The group is `counter & 7`, and the counter lives in the task's **own data** at `+0x8`:

- a run landing on a **day of the year below 10** resets it to `0`;
- any other run increments it by one;
- **if it is above `0x1F` the routine returns immediately** - no captain is looked at, no one
  ages, no administrator gains.

A year holds about 36.5 runs, so the counter normally walks `0` to `~36` and the last handful
of runs in each year do nothing at all.

The counter is part of the saved game, and a scenario can therefore ship with it already past
the cut-off. Measured on the stock campaign starting 1 April 1362: the counter reads **117**
at the campaign's own start date, and because that year's one run inside the reset window
falls before the start date, the next reset is 1 January 1363. For those nine months no
captain in that campaign grows, ages, or gains administrator skill. A save from the same
campaign 259 days after the reset showed 35 records being cut back to their ceilings once the
sweep resumed. An open-ended game starts the counter at `0` and a campaign starting in 1305
was measured resetting every year, so this is a property of the scenario, not of campaigns in
general.

## Interval
Rescheduled by the dispatcher, not by the handler: `due += 0xA00`, so every ten days.
