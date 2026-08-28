# Captain Retirement Freeze

## Summary
When a captain retires, the game can lock up completely: no error, no crash, no report -
one processor core at 100% and a window that stops repainting. The session is lost.

It happens when the captain leaves the ship he was recorded on in the short window between
being flagged for retirement and the retirement being carried out - dismissed, reassigned,
or the ship sunk or sold. That window is one tick for a human merchant's captain, but
**three hours of game time for an AI merchant's**, and AI ships are sunk and sold
constantly. So most of the exposure is other merchants' fleets, which is also why the
freeze looks unprovoked: a player who never touches a captain can still be hit by one
retiring in somebody else's fleet.

The cause is an infinite loop of two instructions in the code that re-finds the captain.

## Details
The ten-day sweep retires an auto trader once he passes
[50 years](../auto-traders/retirement.md), scheduling
[scheduled task](../scheduled-tasks.md) `0x27` - whose handler `0x004DDC00` takes him off
his ship - and setting `field_E` so that a second removal is never queued for the same
captain. The task carries two values in its data union:

|Data|Meaning|
|-|-|
|`+0x0`|the ship index the captain was on|
|`+0x4`|the auto-trader index of the captain|

Both scheduling paths are short-fused: an AI owner's captain at `now + 0x20` (three hours,
`0x004DCFFF`), a human owner's through operation `0x13` at `now + 1` tick (`0x00538C8F`).

### Re-finding the captain
Because he may have moved, the handler first checks whether the recorded pair still holds
(`0x004DDC54`..`0x004DDC60`): the ship index must be in range and that ship's
`field_42` must still name the wanted captain. When it does - every ordinary retirement -
the handler goes straight to its work.

When it does not, it searches, iterating merchants **downwards** from
`[0x006DE4AA] - 1` and taking each merchant's first ship (`merchant+0xE`). The first
merchant whose first-ship index is in range reaches this:

```
4ddc9f: mov  esi,[esp+0x10]        ; the ships array
4ddca3: lea  edx,[eax+eax*2]
4ddca6: shl  edx,0x7               ; eax = that merchant's first ship
4ddcab: mov  cx,[edx+esi*1+0x42]   ; that ship's captain
4ddcb0: cmp  ecx,ebp               ; the captain we want
4ddcb2: 75 fc  jne 0x4ddcb0        ; displacement -4: back to the compare
```

`75 fc` jumps from `0x004DDCB4` back to `0x004DDCB0`. The loop body is the `cmp` and the
`jne`, and neither writes `ecx` or `ebp` - so if the compare fails once it fails forever.
Nothing faults, so no crash handler is entered; the process simply spins.

Two further mistakes in the same six instructions show the block was never exercised:

- **the step to the next ship is missing.** The search should follow
  `field_4_next_ship_of_merchant` along the owner's
  [ship chain](../ships.md#iterating-one-merchants-ships) before comparing again. Nothing
  in the block touches `+0x4`.
- **`0x004DDC9F` destroys the merchant counter.** It loads the ships array into `esi`,
  which is the merchant loop's own down-counter (`0x004DDC6F`, `0x004DDC71`). Even without
  the spin, the merchant iteration would be broken after the first merchant with a valid
  first ship.

The odds of surviving the compare are poor by construction. Counting merchants down means
the first one examined is the highest index, which in a normal game is the human player,
and his first ship is unlikely to be the one carrying the captain who is retiring - a ship
with no captain at all reads `0xFFFF` and can never match. In practice, **entering the
search is entering the hang.**

### What the handler does when the search works
Once a ship index is settled, `0x004DDCCD` switches on the ship's
[status](../ships.md#ship-status) `field_134` through a jump table (`0x004DDE38`, byte
index table `0x004DDE54`, statuses `0`..`0x14`, anything above bails), so what retirement
does depends on where the ship is. Two details of that machinery are worth knowing:

- **the handler can postpone itself.** In the convoy branch (`0x004DDCFC`) it reaches back
  through the scheduled-tasks singleton - `[this]` array base, `[this+8]` the index being
  executed - adds `0x20` to its own due timestamp and returns 1.
- **the return value is the keep-or-drop signal.** `0` at the bail-out, `1` on the paths
  that acted or rescheduled. The dispatcher compares it against `ebp` at `0x004D8A12`,
  which is zeroed at `0x004D85EB`, and `0` takes the path that frees the task.

The bail-out `0x004DDE2D` itself is a pure no-op - `pop`s, `xor eax,eax`, `ret`. It leaves
the captain, the ship and the flag untouched. That is what the game does today whenever
its broken search runs out of merchants without hanging first.

### Reproduced
Forced by scheduling task `0x27` by hand with a deliberately mismatched pair - a ship
carrying no captain, and a captain sitting on another ship. The game froze on the next
dispatcher pass, and the frozen process measured from outside showed `Responding = false`
and **5.06 seconds of processor time consumed in 5.0 seconds of wall clock**: exactly one
core saturated. A deadlock or a wait on a handle would consume almost none. No
`_crash_report.txt` was produced, as expected when nothing faults.

The same task scheduled with a *matching* pair completed normally and took the captain off
the ship, which confirms that the task construction and the rest of the handler are sound
and isolates the fault to the search.

## The fix
`mod-fix-captain-retire-hang` hooks the handler's only call site (`0x004D88EF`) and
resolves the pair before delegating, so the broken search is never entered:

|Case|Behaviour|
|-|-|
|the recorded pair still matches|call the original untouched - it skips the search and works as designed|
|the captain is on a different ship|rewrite the task's ship index, then call the original, which now sees a match and retires him from the ship he is really on|
|the captain is on no ship at all|return `0` without calling the original - the same no-op the game's own `0x004DDE2D` performs, and the value the dispatcher reads as "done, free the task" - and clear `field_E`, see below|

The third case needs one more step, because of `field_E`. Both scheduling paths set it -
`0x004DD023` in the sweep's AI branch, `0x00538CB5` in operation `0x13` - and the sweep
skips a captain whose `field_E` is set (`0x004DCFE0`) outright and forever. So simply
dropping the task would leave the flag standing and make that captain **permanently
unretireable**: he is off a ship now, and the sweep never tests a captain who is off a
ship, but the moment somebody hires him again he would sail on past 50 for good.

The mod therefore clears `field_E` when it drops the task. The flag means "a removal is
already queued"; abandoning the removal has to withdraw that claim, and then the next
sweep reaches him normally if he is ever employed again.

What retirement *does* remains entirely the game's own status-dependent code; the mod only
decides which ship that code is given.

The handler's `due += 0x20` self-postpone was deliberately not reused for a moved captain:
one who has left his ship permanently would be rescheduled every three hours forever -
no freeze, but endless churn.
