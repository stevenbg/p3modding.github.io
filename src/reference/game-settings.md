# Game Settings
The choices made on the new-game screens live in one object, reached through the pointer at
`[0x006CC3E8]`. Difficulty is a **preset** rather than a setting of its own: `+0x0E` stores
which one was picked, and the individual settings it stands for are stored separately and can
each be set on their own.

The block is initialised wholesale by the run of stores from `0x00463AEE` to `0x00463B89`.
It writes `bl` into the dword at `+0x4` and into the bytes `+0x9`, `+0xD`, `+0xE`, `+0xF`,
`+0x10`..`+0x16`, `+0x18` and `+0x19`, and forces `+0xA`, `+0xC` and `+0x17` to `1`. `+0xB`
is the one byte in that span it never touches. That is a reset to defaults, **not** the code
that applies a difficulty preset - it would leave every setting holding the same value, which
is not what the presets produce.

## The Settings Screen Writes the Block
The block is filled in from the new-game settings screen, whose controls are **steppers** - a
label with `-` and `+` beside it that cycles through the values. One routine copies nine of
the screen's fields into the block at `0x00499304`..`0x0049938D`, seven of them with a `dec`,
which is where the screen's 1-based values become the 0-based stored ones:

|Settings byte|Copied from|`-1`|Site|
|-|-|-|-|
|`+0x04` (dword)|`window + 0x1BAC`|no|`0x0049930A`|
|`+0x09`|`window + 0x1B98`|yes|`0x0049931A`|
|`+0x0D`|`window + 0x1BB4`|no|`0x00499329`|
|`+0x10`|`window + 0x1B80`|yes|`0x0049936C`|
|`+0x11`|`window + 0x1B84`|yes|`0x0049933A`|
|`+0x12`|`window + 0x1B8C`|yes|`0x0049937C`|
|`+0x13`|`window + 0x1B90`|yes|`0x0049935B`|
|`+0x16`|`window + 0x1B94`|yes|`0x0049934A`|
|`+0x17`|`window + 0x1B88`|yes|`0x0049938D`|

The seven that take the `-1` are **exactly** the seven bytes measured below as varying with
difficulty, and their sources are seven consecutive dwords of an eight-entry array at
`window + 0x1B80`. That array is loaded with each entry clamped to **1..3** (`0x00497E1E`,
`0x00497E39`), except the eighth, which is forced back down to `2` at `0x00497E54` - a
two-position control, and the one entry of the array this routine does not copy. The other two
fields, `window + 0x1BAC` and `window + 0x1BB4`, lie outside that array and take no `dec`.

Three positions on the screen, values `0`..`2` in the block: that is the whole of what the
seven difficulty settings are. What moves those controls when the difficulty itself is changed
has not been found.

|Offset|Meaning|
|-|-|
|`+0x0D`|which folder the game saves to, from the path builder at `0x005473A6`: `5` = `Save\Kam` (campaign), `3` = `Save\Ein` (single open-ended), `0`..`2` = `Save\Mehr` (multiplayer)|
|`+0x0E`|the difficulty preset, `0`..`4` from easiest to hardest|
|`+0x13`|**Pirates activity**, `0` low, `1` normal, `2` high - the byte world generation reads to decide how many [pirate bands](../pirates/bands.md) exist|

The individual settings that vary with the preset, measured by starting one game at each of
the five presets:

|Offset|preset 0|1|2|3|4|
|-|-|-|-|-|-|
|`+0x09`|0|1|1|2|2|
|`+0x10`|0|0|1|1|2|
|`+0x11`|0|0|1|1|2|
|`+0x12`|0|0|1|1|2|
|`+0x13`|0|1|1|2|2|
|`+0x16`|0|1|1|2|2|
|`+0x17`|0|0|1|1|2|

Every other byte in the block was identical across all five, so those seven plus the preset
index at `+0x0E` are the whole of what difficulty changes.

The seven fall into just two columns, and both are a halving of the preset index `p`:

|Column|Settings|Values|
|-|-|-|
|`(p + 1) / 2`|`+0x09`, `+0x13`, `+0x16`|`0, 1, 1, 2, 2`|
|`p / 2`|`+0x10`, `+0x11`, `+0x12`, `+0x17`|`0, 0, 1, 1, 2`|

So the five presets walk the two columns half a step out of phase, across steppers that only
have three positions each.

## What Difficulty Does Not Affect
Town [production](../towns/production.md) is untouched by it. Five games started at the five
presets produced identical facility efficiencies and an identical productivity distribution
across all 24 towns, so neither the effective/low grade of a ware nor the output of a facility
depends on the difficulty.
