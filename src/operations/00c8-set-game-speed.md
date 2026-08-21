# Set Game Speed
Operation `0xC8` changes the [game speed](../basics/time.md#game-speed) - every
speed control ends up enqueueing it.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0xC8`|
|0x04|u32|ms per tick of mode 0 (`operations+0x8D4`) - the speed slider's divisor|
|0x08|u32|ms per tick of mode 1 (`operations+0x8D8`)|
|0x0C|u32|the pacing mode itself (`operations+0x92C`)|
|0x10|u32|stored to `operations+0x91C`, purpose unknown|

A field of `-1` leaves that setting unchanged. Like
[Advance Time](./00c4-advance-time.md) this opcode is handled inline by
`execute_operations` (handler `0x00546DCF`, jump table `0x00547290`), not by the
operation switch.

The handler also sets the master run flag `operations+0x914` to 1 - unless a
network round is pending (`operations+0x928` nonzero), in which case it pauses
instead (run flag 0, mode 0). The mode-0 divisor passes through extra clamp logic
(`0x00546DFA`..`0x00546E36`, a floor derived from the global `[0x0066DE7C]`, not
fully decoded) before landing in `+0x8D4`; a change of mode resets the pacer's
last-advance timestamp (`operations+0x938`).

Known enqueuers: the fast-forward button (`0x00420300`, mode 1) and its
counterpart (`0x004202A0`, mode 0), both leaving the divisors at `-1`; the speed
slider, which passes a real mode-0 divisor; and assorted game flows that force
mode 0 (e.g. `0x00433865`).
