# Retirement
An [auto trader](../auto-traders.md) record has a lifespan. `field_4` is a **birth stamp**
in ticks, not an age: the initializer writes `game_time - offset` with

```
offset = 46720 * (48..79) + 1792 * (0..31)
```

so a newly created record is between 24.0 and 40.1 years old. Age is therefore
`game_time - field_4`, and it grows on its own without anything having to maintain it.

## The Age Test
The [ten-day sweep](../scheduled-tasks/0003-ten-day-update.md) retires a captain once his
age passes `0x474A00` ticks = 18,248 days, almost exactly **50 years**, and only while
`field_E` is still clear. That flag is what stops a second removal being queued for a
captain already on his way out.

Which path runs depends on the owner:

- an **AI** merchant's captain is retired at once - the sweep schedules task `0x27`
  (`0x004DDC00`) three hours out and sets `field_E`. No operation and no message are
  involved.
- a **human** player's captain gets a roll instead: `(rand & 0x3FF) * (age >> 13)`, floored
  to a multiple of 1024, must exceed `0x93000`. The product cannot clear that bar until the
  captain is about **51.6 years** old, and the chance grows from there. When it fires it
  enqueues [operation `0x13`](../operations/0013-captain-retirement.md), which schedules the
  same task and sends the player a message.

Task `0x27` is what actually takes the captain off his ship - and it can freeze the game
outright if the captain has moved since it was scheduled, see
[Captain Retirement Freeze](../bugs/captain-retirement-freeze.md).

## Nothing Retires in a Tavern
The sweep only walks merchants' ship chains, so a record waiting in a tavern is never
tested - however old it is. Ageing, like [skill](./skill.md), only happens to a captain
somebody employs.
