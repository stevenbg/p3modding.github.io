# Notification Tickers
The popup boxes on the scrollmap - events on the top left ("Game speed: ..."),
incoming-letter notices on the top right ("Trading information: ...") - are two
queues on one manager object, held in the static `0x006CBB40`.

|Field|Meaning|
|-|-|
|`+0x168`|left-queue slot widgets, stride `0xA0`, text object at slot `+0x9C`|
|`+0x488`|left-queue count (byte, capacity 5; enqueue bails when full)|
|`+0x494`|right-queue slots, stride `0xA0`|
|`+0x7B4`|right-queue count (byte, capacity 5)|
|`+0x7D0`|left-queue expiry ticks, one u32 per slot: enqueue tick + `0x2EE0`|

## Posting
- Left/event popup: `0x0042B6A0` (thiscall(this, text)) - takes a **plain C
  string** and does everything: picks the slot, sets the text, stamps the expiry
  (12000 ticks from `[0x006DCCF0]`, the tick counter). Anything can post one.
- Right/letter popup: `0x0042BB20` (thiscall(this, string)), followed by a
  `0x004237D0` refresh - what the letter announcer uses.

## The Letter Announcer
When a letter is delivered to the player, the mailbox insert (`0x004D6680`)
dispatches by category (byte table `0x006C0198[type]`, jump table `0x004D6760`)
and, gated by the mailbox's notification settings, calls the announcer
`0x004D7B10`:

- mailbox `+0x26` is the notification bitmask (the in-game message options):
  bits 0-2 play the arrival sound per category (`0x00443150`, sound id `0x1770`),
  bits 3-5 post the popup per category.
- The popup text is `sprintf(template, type_name)`: templates at
  `[0x006A4570]`/`[0x006A4574]`/`[0x006A4578]` per category ("Trading
  information: %s", ...), type names from the string table at `0x006A52D0`
  indexed by message type; the scripted letter types `0x3C..0x40` use their
  letter-text payload instead.
- mailbox `+0x38` (values interpreted through the table at `0x006C0220`) gates
  whether the announcer runs at all.

See [Letters](../letters.md) for the message structures.
