# Administrators
An office administrator is an [auto trader](../auto-traders.md) record like any other, but
almost nothing a captain does applies to him. He never sails and never fights, so his
navigation and combat bytes are dead stats: **only trade does anything for him**, and what
it does is the [buying discount](./skill.md#the-buying-discount).

## He Gains on a Path of His Own
Administrators are grown by the same [ten-day sweep](../scheduled-tasks/0003-ten-day-update.md)
that grows captains, but through
[operation `0x67`](../operations/0067-administrator-skill-gain.md) rather than operation
`0x12`. The producer computes `new = (trade + 43) & 0xFF` and drops the whole thing when
that would wrap below the current value. A fresh administrator starts at `0`, so his trade
skill walks exactly

```
0, 43, 86, 129, 172, 215
```

and then stops - always an exact multiple of 43, which is exactly one displayed level. Only
a **human** merchant's administrators gain at all.

Because there is no [ceiling table](./skill.md#the-ceiling-belongs-to-the-slot) anywhere on
this path, there is also nothing to lose to a clamp, and the outcome is not in doubt:
**every administrator left in place long enough reaches level 5.** Only the pace is random.
The producer rolls `(rand & 0x3FF) < 0x1B3` on each of his eligible rounds, so about 42% of
them pay - roughly two eligible rounds per level.

## Dismissing One Costs You the Man
Unlike a captain, an administrator does not go back into the tavern to be re-hired: his
record is freed to the freelist and the hire path allocates a fresh one at trade `0`. See
[Array Layout and Slot Recycling](../auto-traders.md#array-layout-and-slot-recycling) for
the mechanism.

The consequence in play: a level 5 administrator you dismiss is gone, and his replacement
starts from nothing and has to climb all five levels again. The wage the window quotes for
the vacant post is the level 0 cost, not a memory of the man who left - see
[what the interface shows](../auto-traders.md#what-the-interface-shows-is-not-the-records-wage).

## The Administrator's Trading
An administrator is not driven by the [ships tick](../ships.md) like a captain on a
[route](../ships/trade-routes.md), but by the world tick itself:

```
advance_time 0x00530E80
  └─ 0x0051BA10   walk the town's offices, chaining office+0x2CA
       └─ 0x004FFF20   the per-office periodic routine, dispatching on office+0x2D6:
            bit 0x10 -> 0x004FFA30   AI-merchant offices (skipped when merchant+0x8 is 0)
            bit 0x02 -> 0x004FFC20
            bit 0x01 -> 0x004FF780   the administrator's trading
```

The town whose offices are visited comes from the tick counter (`tick >> 3`), so offices are
worked through in a staggered round rather than all at once.

`0x004FF780` walks the wares from 23 down to 0 and acts on each
[order](../merchants/trading-office.md) whose price is non-zero. On the sell side (positive
price, the minimum price) it offers the stock **above** the minimum store quantity - so that
column is a floor the administrator sells down to, not a target it tops up to. Purchases
apply the administrator's own [buying discount](./skill.md#the-buying-discount).
