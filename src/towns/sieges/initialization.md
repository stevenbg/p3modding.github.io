# Initialization
Sieges are started by operation type `0x8E`, the `handle_start_siege` function is at `0x00633AF0`.

## Army Size
The three compositions live at `0x0067B5C8`, three rows of five dwords:

|Row|Swords|Bows|Crossbows|Carbines|Trebuchets|
|-|-|-|-|-|-|
|0|262|131|99|0|55|
|1|262|131|0|87|55|
|2|262|0|99|87|55|

The row is selected by `lea ecx,[ecx+ecx*4]` then `lea ebp,[ecx*4+0x67b5c8]` at `0x00629E7C`,
from the low byte of the first argument of `0x00629C60` - the routine `handle_start_siege`
calls at `0x00633C30`. Whether the siege type the operation carries **is** that value or is
remapped on the way has not been traced, so the rows are numbered as rows here rather than by
siege type.

These are not the counts that reach the field. Each entry is multiplied by a runtime scale and
stored as the high word of a 16.16 value before the clamp
(`imul ecx,[esp+0x54] / add ecx,0x7fff` at `0x00629E89`), so what a siege actually brings is
`(entry * scale) >> 16`, then clamped to the following values at `0x0067B604`:

|Squad|Limit|
|-|-|
|Swords|40|
|Bows|22|
|Crossbows|18|
|Carbines|15|
|Trebuchets|6|

## Gate and Ram
Both the gate and the battering ram have 0x10000 (65536) HP.

## Approach

