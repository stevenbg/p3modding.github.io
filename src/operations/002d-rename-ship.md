# Rename Ship
Operation `0x2D` renames a ship - the shipyard's "change name" button. Operation
`0x2E` appends to the name: the shipyard sender (`0x005FAF5D`) chunks the entered
name by 12 characters, sending the first chunk as `0x2D` and every further chunk
as `0x2E` (its text field allows 15 characters total). Both share the layout:

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x2D` (set) / `0x2E` (append)|
|0x04|char[12]|name chunk, latin1, NUL-padded|
|0x10|u32|ship index|

The switch case (`0x00535CAA`) forwards to a handler at `0x0053CCE0` that is shared
by a family of rename operations. For `0x2D` it copies exactly 12 name bytes into a
stack buffer (forcing a NUL terminator after them - names are at most 12
characters), bounds-checks the ship index against the ship count (`0x006DD894`,
ships array `[0x006DD7A4]`, stride `0x180`), and assigns the name through the
dynamic-name registry (`0x00512D40`, this = `0x006DDAA0`, see
[Name Banks](../ui/name-banks.md)) targeting `ship+0x15E` - the ship's registry id
word. That call keeps both the registry (which letter texts resolve ship names
from) and the ship's inline name buffer at `+0x160` in sync. The `0x2E` branch of
the same handler appends its chunk to the ship's existing registry name instead
(via `0x00512FC0`) rather than replacing it.

Captured example, renaming ship `0x60` to "Hunter 31":

```
2d 00 00 00 48 75 6e 74 65 72 20 33 31 00 00 00 60 00 00 00
```
