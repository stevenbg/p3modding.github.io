# Time
The game time is stored in the static `game_world` struct at offset `0x14` as *ticks*, and is increased by the `advance_time` function at `0x00530E80`.
That function has exactly one caller: the inline handler of the
[Advance Time operation](./operations/00c4-advance-time.md) - game time only ever
advances through the operation queue.

## Ticks
Every ingame day is 256 ticks long, so there are 93440 ticks in a year.
Consequently the least significant byte conveniently encodes the time of day.

## The Calendar
The day, month and year are not counted up as time passes - they are **derived from the
tick counter once a day** by `0x005310D0`, so nothing else has to keep them in step:

|Field|Meaning|
|-|-|
|`game_world+0x0`|day of the month, 1-based|
|`game_world+0x1`|month, 0-based|
|`game_world+0x2`|year (u16), `ticks / 93440`|
|`game_world+0x4`|day of the year (u16), `(ticks >> 8) % 365`, so 0-based|

The routine writes the year and the day of the year straight from the counter, then walks
the month table at `0x00672D78`/`0x00672D7A` - the cumulative day of the year at which each
month starts and ends - to find the month, and subtracts that month's start to get the day
of the month.

The day of the year is read by game logic, not just by the interface: the
[ten-day update](./scheduled-tasks/0003-ten-day-update.md) resets its round counter on any
run that lands in the first ten days of a year.

## Ticking Objects
Different game objects tick at different intervals.
Information about what happens in those ticks can be found in the respective chapters.

### Towns
The town tick handler is `handle_town_tick` at `0x0051BA10`. A town ticks if one of the
following equations is true:
```c
game_time & 0b111 == 0b011 &&
town_index == (((unsigned __int8)game_time) + 255) >> 3
```
```c
game_time & 0b111 == 0b111 &&
town_index == ((unsigned __int8)game_time) >> 3
```
This results in the following town tick behaviour:

|Town Index|Game Time LSB|
|-|-|
|32|0b00000_011|
|33|0b00001_011|
|34|0b00010_011|
|35|0b00011_011|
|...|...|
|39|0b00111_011|
|00|0b00000_111|
|01|0b00001_111|
|02|0b00010_111|
|03|0b00011_111|
|...|...|
|31|0b11111_111|

#### Facilities
All facilities tick when their town ticks.

## Game Speed
The tick pacer inside `execute_operations` (`0x00546640`) converts real
milliseconds into [Advance Time operations](./operations/00c4-advance-time.md)
once per frame. How many ticks a batch gets depends on the pacing *mode*
(`operations+0x92C`) and its ms-per-tick divisor:

|Mode|What|ms per tick|Max ticks per batch|
|-|-|-|-|
|0|normal play|`operations+0x8D4` - set by the speed slider|8|
|1|fast forward (the mode with its own window)|`operations+0x8D8` (2 in vanilla)|256 (a day)|
|2|local map (town view, sea battle)|the constant `[0x00673CF8]` = 3375|1|

The six positions of the speed slider set the mode-0 divisor to 3515, 468, 351,
234, 117 and 78 ms per tick (measured in vanilla 1.1); the slider never changes the
mode. Entering the local map switches to mode 2, whose pace is a hard constant -
which is why the speed controls have no effect there. All speed changes travel as
the [Set Game Speed operation](./operations/00c8-set-game-speed.md), and
`operations+0x914` is the master run flag the pacer requires (0 = paused).

The same pacer also enqueues the autosave operation (`0xC2`) whenever the timer at
`operations+0x940` expires (period `operations+0x944`, 180000 ms in vanilla).

## The Frame Clock
Real time reaches the game through one updater (`0x004BD180`), called once per
frame from the main loop's frame function: it reads the OS time, sleeps the
remainder of a 20 ms frame (the game is capped at 50 fps, `0x004BD1A9`), and
computes the frame's elapsed milliseconds into two globals:

|Global|Meaning|
|-|-|
|`0x006DCCF0`|raw OS time of the last frame|
|`0x006DCCF4`|this frame's elapsed ms|
|`0x006DCCF8`|accumulated game clock (ms), the sum of all frame deltas|

The tick pacer measures against `0x006DCCF8`, and so does the local-map simulation
(ship movement, projectiles, battle AI): it is paced neither by game ticks nor by
how often its update runs - calling the update several times per frame moves
nothing - but purely by this clock. Scaling the delta before it is stored (its
computation at `0x004BD1E2` is a detourable 5-byte sequence) therefore speeds up
the local map and the world alike; mod-ui-tweaks' "extra speed" does exactly that.
