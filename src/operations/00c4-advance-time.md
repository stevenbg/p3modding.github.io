# Advance Time
Operation `0xC4` advances the game time - the ONLY way it advances: its handler
makes the executable's single call to `advance_time` (`0x00530E80`, see
[Time](../basics/time.md)).

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0xC4`|
|0x04|u32|from: the current game tick|
|0x08|u32|to: the tick to advance to|

Unlike the low operations, `0xC4` never reaches the operation switch: opcodes
`0xC1..0xD4` are handled inline by `execute_operations` through its own jump table
at `0x00547290`, and `0xC4`'s inline handler at `0x00546A1C` calls
`advance_time(game_world, from, to)` directly.

The operations are produced by the tick pacer inside `execute_operations`
(`0x00546640`): once per frame it converts the real milliseconds elapsed since the
last advance into a tick count according to the current
[game speed](../basics/time.md#game-speed), and enqueues one `0xC4` for the batch.
The same pacer enqueues the autosave operation (`0xC2`) whenever the autosave timer
(`operations+0x940`, period `operations+0x944` - 180000 ms in vanilla) expires.
