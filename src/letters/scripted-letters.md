# Scripted Letters
Mission and event letters (escort/patrol updates, town news, notifications with a
letter body) are produced by a letter script interpreter: a bytecode stream of
commands with byte operands, executed against an array of script variables
(interpreter object `+0xC`). One of its commands creates and sends a complete
letter; its handler starts at `0x004ED4A0`.

## The Create-Letter Command
The handler allocates a 16-byte [message](../letters.md) and fills it from the
command's operands (`cmd[n]` below) and the script variables (`var[n]`):

|Field|Value|
|-|-|
|type (`+0x4`)|`0x3C + cmd[1]`; kinds past `0x40` become type `0x71` (`0x004ED4EA`)|
|town (`+0x5`)|the **low byte** of `var[cmd[2]]`, unvalidated (`0x004ED4E4`)|
|descriptor (`+0x8`)|`malloc(0x10)`, filled from `cmd[3..6]` and further variables|
|text (`+0xC`)|`malloc(0x1000)`, the formatted letter body|

The scripted letter types `0x3C..0x40` are the ones the letters list and the
[notification tickers](../ui/notification-tickers.md) treat specially: their
display name comes from the letter-text payload instead of the type-name string
table at `0x006A52D0`.

## Text Formatting
The letter body is built by a `%`-substitution engine inside the same
interpreter: it copies the template text, expanding placeholders from replacement
string tables (e.g. `0x006C3040`) and game data - among them the dynamic-name
resolver `0x00512B20` (ship and similar names, see
[Name Banks](../ui/name-banks.md)). The names in the letter text are resolved
from their own sources at creation time, so the text is correct even when the
message's town byte does not hold a town - the two come from different places.
Descriptor`+0x0` holds the finished text's length (the creation code tracks the
raw write position there while building), which bounds the text.

## Sending
The handler ends in [add_message](../letters.md) calls: to a single recipient
resolved from a script variable (`0x004EDEAA`, merchant = `var[cmd[5]]`), or in
broadcast loops over every merchant (`0x004EDE30`, and a variant at
`0x004EDDA4`), honoring the mailbox gate word.

## The Town-Byte Flaw
Because the town byte is the unvalidated low byte of an arbitrary script
variable, letter templates whose variable is not a town index - the
escort/patrol mission's "Patrol destination" letters pass one that reaches values
like 40, 95, 228 or 255 - send letters whose town byte is garbage. The letter
itself is fine; the letters list's town column is not. See
[Patrol Letter Crash](../bugs/patrol-letter-crash.md).
