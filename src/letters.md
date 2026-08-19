# Letters
Every message a merchant receives - the personal letters, town announcements,
mission updates - lives in one global message pool, and the letter windows are
views over it.

## Message Pool
The pool object sits at `0x006DD730`: `+0` holds the pointer to the entry array,
the word at `+0x6` (`0x006DD736`) the current pool size (it grows on demand,
`0x004D6B40`). Entries are 16 bytes:

|Offset|Meaning|
|-|-|
|`+0x0`|day of the letter's date|
|`+0x1`|bit 7: unread flag; low nibble: month index (displayed month is nibble + 1)|
|`+0x2`|year (u16)|
|`+0x4`|message type byte (valid types are `< 0x86`; free entries hold `0xFF`)|
|`+0x5`|town byte, drawn as the letter list's town column (see the [bug](./bugs/patrol-letter-crash.md))|
|`+0x6`|index of the next message (u16) - the per-merchant chain, or the freelist for free entries|
|`+0x8`|payload; for [scripted letters](./letters/scripted-letters.md) a pointer to a 16-byte descriptor|
|`+0xC`|payload; for scripted letters a pointer to the letter text|

## Mailboxes
Messages are chained per merchant. The mailbox manager object at `0x006DE4A0`
holds the current date at `+0x0` (day), `+0x1` (month) and `+0x2` (year, u16) and
the merchant count at `+0xA` (`0x006DE4AA`); `0x005303C0` (thiscall(this =
`0x006DE4A0`, merchant)) returns a merchant's mailbox record, whose word at `+0xA`
is the head message index of that merchant's chain. The word at `+0x8` acts as a
gate: broadcast deliveries skip merchants whose gate is nonzero.

## Adding Messages
`add_message` at `0x004D6530` (thiscall(this = `0x006DD730`, merchant, message*))
takes a caller-prepared 16-byte message - the caller fills type, town and
payloads - then stamps the current date, sets the unread bit, allocates a pool
slot and links it into the recipient's chain. A merchant argument of `-1`
broadcasts the message to every merchant (skipping gated mailboxes); `-2` and
`-3` take special paths that are not fully mapped (`-3` resolves a recipient
through an office). The function has over 150 call sites - one per message kind -
each preparing its own struct.

## Display
The letters window groups messages into its four tabs through the byte table at
`0x006C0198`, indexed by message type. Unread messages draw black
(`0xFF000000`), read ones brown (`0xFF5A2406`). See
[Personal Letters Window](./ui/personal-letters-window.md) for the window
internals.
