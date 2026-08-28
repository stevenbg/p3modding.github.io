# Church Window Animation Crash

## Summary
Opening the church shortly after loading a save can crash to desktop: a write through
a NULL pointer at `0x0046B49D`, inside the
[shared interior animation player](../ui.md#building-interior-animations)'s
frame loader. The church window's tick asks the player to stream "the next frame of
my loop animation" without checking that the player has that animation - or any -
loaded, and on a freshly recreated player the frame array is NULL.

The trigger needs a load **through the main menu** (quit to menu, then load). An
in-game load keeps all UI objects alive, animation player included, so the same
stale window state meets a player whose array is still allocated and vanilla merely
plays the loop - which is why the crash resists casual reproduction.

Captured twice in the wild on consecutive attempts (identical registers both times:
player `+0x38` = `0xFF`, array NULL, requested animation 6).

## The window's state machine
The church window (static `0x006E556C`, 0x1D60 bytes, constructor `0x005C88E0`)
carries:

|Field|Meaning|
|-|-|
|`+0x1D30`|mode: -1 idle, 1..3 the donation pages, 4/5 the post-donation "monks working" states. **No writer in the constructor.**|
|`+0x1D38`|town index|
|`+0x1D5E`|church stage, from `0x004FE360` on `town+0x794`; > 4 selects the with-altar animation pair 8/9, otherwise 6/7|
|`+0x1D5F`|play-the-thanks-animation flag: set on donation (`0x005CB24D`), cleared when the one-shot completes (`0x005C9621`)|

Mode writers: the hide handler `0x005C94AC` (-1), set_mode `0x005C9DD1` (its
argument), the donation flow (`0x005CB08C` = 4, `0x005CB7A3` = 5). set_mode's
epilogue (`0x005CA166`) is written correctly: whenever mode > 0 it calls
switch_animation unless the right animation is already current.

The tick (`0x005C956D`), however, decides by elimination:

```
;; wants the loop animation (ebp = 6 or 8); esi = the thanks animation (7 or 9)
005C962A  xor  eax,eax
005C962C  mov  al,[ecx+0x38]     ; the player's current animation
005C962F  cmp  eax,esi           ; "is the THANKS animation current?"
005C9631  jne  0x5C963C          ; no -> assume the loop is loaded, stream a frame
005C9633  push edi
005C9634  push ebp
005C9635  call 0x0046B710        ; yes -> switch to the loop properly
005C963C  push ebp
005C963D  call 0x0046B110        ; load_next_frame - writes through the NULL array
```

"Not the thanks animation" is also true of a fresh player (`current = 0xFF`) and of
a player holding another building's animation - the first crashes, the second
streams church frames into an array sized and cursored for a different animation, a
silent heap overwrite.

## How a stale mode arms it
The window is recreated on every menu-path load, but its constructor never writes
`+0x1D30`, and the new window usually mallocs into the LFH block the old one vacated
- so the mode is **recycled from the previous session**. The open path only calls
set_mode when a specific page was requested: the tab-request byte (`+0x4D2` of the
tab controller) is `0xFF` after any previous open, and `0xFF` skips both set_mode
calls (`0x005A5BDD`). Result: leave the church in an animating state (any donation
page, or monks working), quit to the main menu, load, click the church - the first
tick runs with the recycled mode > 0, the thanks flag 0, and a fresh player, and
takes the `load_next_frame` branch.

The tavern window shows the intended pattern: all fourteen of its stream calls sit
behind an explicit current-animation check, mismatches go through switch_animation,
and its teardown call is followed by resetting the player's `+0x38` by hand
(`0x005CD503`). The church tick is the one caller in the game that infers instead
of checking.

## The fix
`mod-fix-church-anim-crash` patches three bytes in the tick so that "not my loop
animation" - fresh player, foreign animation, or the thanks handover - routes
through switch_animation, which allocates, loads frame 0 and positions the sprite:

|Address|Original|Patched|Effect|
|-|-|-|-|
|`0x005C95E2`|`jne` rel8 `0x59`|rel8 `0x47`|the thanks-branch mismatch falls into the loop dispatcher instead of streaming raw|
|`0x005C9630`|`cmp eax,esi`|`cmp eax,ebp`|compare against the loop id, not the thanks id|
|`0x005C9631`|`jne`|`je`|equal streams the next frame, anything else switches|

All six mode/flag/current combinations were enumerated: every previously working
path is unchanged, and the two broken ones now play the loop animation. As belt and
braces the mod also detours the player's three unguarded methods to bail out when
the frame array is NULL.
