# Alderman Election

Scheduled task opcode `0x20`, handler `0x004DBBD0`. Created once at world
initialization (`0x004D8B60`) for `now + 0x5A00` - **game day 90, at midnight** - and
re-armed by its own finale for `+0x16D00`: **one election per 365 days**.

## Election day is a tick-stepped state machine

The handler branches on its own due time's **time-of-day byte** and advances the same
task by single ticks (`0x004DBEEB`, `0x004DBF20`), so one election runs as a sequence
of phases across a single day:

- **midnight (byte 0)**: four candidate bytes are picked into the task data
  (`+0x8..0xB`, via `0x00529700`; an invalid pick is flagged with `0x80`), and the
  sitting alderman's town (`[0x006DE52E]`) is stamped into `+0x17`;
- **daytime (byte < 0xC0)**: an elimination tournament compares the candidates'
  standing bytes (`+0x10..0x13`), re-firing tick by tick;
- **evening (byte >= 0xC0)**: a further phase at `0x004DBF01`;
- **the last ticks (byte >= 0xFC)**: the finale, `0x004DBF36`.

## The finale

- The winner resolves to a **town** - whose mayor (`town + 0x6F1`) becomes the new
  alderman - or, `0x80`-flagged, to a merchant via his hometown. The alderman index is
  written to `[0x006DE52D]` and his town to `[0x006DE52E]` (`0x004DC11C`,
  `0x004DBFBB`/`0x004DBFCC`); if the resolved mayor is invalid the old alderman stays.
- A **human** winner is notified with note subtype `0x11`.
- On a change of alderman, every town's reputation figures are recomputed
  (the `0x004F75F0` loop).
- **Every councillor bribe in the world is voided**: the reset at `0x004DBF6F` writes
  `0xFFFFFFFF` over `town + 0x6DC` - all four councillors - for every town. Standing
  [bath house bribes](../operations/0042-bath-house-bribe-success.md) do not survive an
  election.
- The task reschedules itself: `due += 0x16D00`, time-of-day zeroed.
