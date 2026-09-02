# Event Window

The window that announces world events - the ones with videos in `Videos\` - is the
singleton at `[0x006CC7E8]`. An announcement is **two calls in sequence** at each raise
site: `0x00469380` opens and shows the window (registering it with the notification
manager `[0x006CBB40]` at `0x004693ED`), then `0x00469AF0`
(`thiscall(this, type, a, b, text1, text2)`) fills it in. Suppressing only the second
leaves a visible window whose type field still holds its idle `-1`, and the renderer
indexes `window+0x3BC` with it unchecked (`0x0046760C`) - a crash.

## Event types: decode the cases, not the tables

Two message string tables exist - plain text at `0x006A3BD0` and the ticker's `\c\f4_`
variants at `0x006A3808` - and **neither is indexed by the event type**. The type
dispatches through the jump table at `0x0046AB50`, and each case loads its own string
pointer; the case order and the table order agree only up to `0x02`. Reading a type's
meaning off the table gives a plausible, self-consistent, wrong answer.

The types, decoded from the cases (the middle of the range):

|Type|Event|Raised at|
|-|-|-|
|`0x07`|Discovery of America|`0x00531A96`|
|`0x09`|Great celebration in %s|`0x004E2605`|
|`0x0D`|Hulk finished|`0x0050860C`|
|`0x0E`|Wedding with %s %s|`0x004E819D`|
|`0x10`|Competitor achieved the objective|`0x004E4EDD`, `0x004E55DD`|
|`0x11`/`0x12`/`0x13`|Famine / Plague / Fire in %s|`0x00527FFC` / `0x004E9432` / `0x004E9A38`|
|`0x14`|**%s has struck again** (a notorious pirate robbed a non-player ship)|`0x0060EBA7`|
|`0x15`|Siege in %s|`0x00629D36`|
|`0x16`, `0x17`|Blockade in %s|`0x004F0EF0`, `0x004E0E0E`|

Types `0x00`..`0x05` are the rank announcements and `0x0A`..`0x0C` the other ship classes.
The table entry `0x006A3C2C` (its string "Pirates are attacking %s" at `0x006A3AC4`) has
**no loader in either table** - a dead string with no event behind it.

The pirate event (`0x14`) is raised only when the robbed merchant is not the player
(`0x0060EB51`), and at most once per 30 game days: `[0x006E59D8]` holds the last raise
time, and the re-arm test at `0x0060E9B3` is `stamp + 0x1E00 < now`.

## Video on, video off - and the fast-forward reset

`0x00469AF0` reads the Options screen's **Event videos** flag (`settings+0x26`, from
`[OPTIONS] VIDEOS` in `P2.CFG`, default on) at `0x00469B3A`. On, the event plays its
video. Off, the case formats a message, posts it on the
[event ticker](./notification-tickers.md), and falls into a shared tail that enqueues
`SetGameSpeed { level: 0 }` at `0x0046A26C` - guarded only by "is a game session live"
(`settings+0xD >= 3` at `0x0046A237`), so **every** event that reaches the text path drops
the game out of fast forward, with nothing distinguishing one type from another.

The window's own type field at `+0x3C0` is written **only on the video path**
(`0x00469BAD`, gated on `+0x3EA`); on the text path it keeps its idle `-1`. Code that
needs the type of a ticker-path event must take it from the announcement call itself.

## Not the letter system

The letter/message events funnelled through `0x00548CA0` (60 call sites) are a separate
system with their own `0x00`..`0x2A` type space (formats at `0x006B28A8`, indexed by type
directly) and their own way of stopping the clock: they zero the speed level and master
run flag on the spot (`0x00548CE3`) and park the event in an 8-slot pending table at
`ops+0x948` (stride `0xA`), drained by `0x00549040`, which restores fast forward when the
last slot clears. The two systems overlap in type numbers but share nothing.
