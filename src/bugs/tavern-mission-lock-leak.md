# Tavern Mission Lock Leak

## Summary
Looking at a mission in a tavern's side room locks the offer to the viewing merchant, so
that nobody else can take it. Leaving the side room by switching to another tavern page
releases the lock; closing the tavern window outright - a right click - does not. The
offer stays locked until it is re-issued, and a locked offer is invisible to every other
merchant.

Single player never notices, because the side room accepts an offer locked to the asking
merchant himself. In multiplayer the leak denies the mission to the other players, and one
player can leak a lock in every town by opening each side room and right-clicking out.

## The Lock
A side room offer is a [tavern mission](../letters/71-tavern-missions.md) letter whose
scheduled task holds the mission's script variables; the variable named by the letter's
`descriptor+0xC` is the lock, holding a merchant index while locked and `0xFFFFFFFF` (or
`0xFFFF`) while free.

The lock is taken and released by the
[tavern interaction](../operations/0052-tavern-interaction.md) operation. Two of its types
matter here, and they work in completely different ways:

- **Type 9, the side room** (handler `0x0053C7A3`) carries the task index and the variable
  slot in the operation itself, at `+0x4` and `+0x6`. With a valid merchant it writes that
  merchant into the variable if it is still free (`0x0053C808`); with an invalid merchant
  index it writes `0xFFFFFFFF` back (`0x0053C7F0`). The panel sends the valid-merchant form
  when a page is opened and the invalid-merchant form when it is left, so the lock is taken
  and released as the player navigates.
- **Type 10, "Leave"** (handler `0x0053C619`) has no task index to work from and instead
  walks the merchant's letter chain with `0x004D7900` to find his offers in that town. This
  is the path a closing tavern window relies on, and it is broken.

## The Defect
Both the entry and the continuation of that search compare the **letter index** against
the **merchant count** at `0x006DE4AA`, where the letter pool size at `0x006DD736` is
meant:

```
0053C6EF  and  eax, 0xffff          ; letter index that 0x004D7900 found
0053C6F4  mov  cx, [0x006DE4AA]     ; merchant count  (should be [0x006DD736])
0053C6FB  cmp  ecx, eax
0053C6FD  jbe  0x0053C80A           ; bail when index >= merchant count
```

```
0053C783  call 0x004D7900           ; next offer in the chain
0053C78F  mov  cx, [0x006DE4AA]     ; merchant count  (should be [0x006DD736])
0053C796  cmp  ecx, eax
0053C798  ja   0x0053C706           ; loop only while index < merchant count
```

A game has a few dozen merchants and a letter pool of hundreds of entries (400 in the save
below), so any offer sitting past the first few dozen pool slots fails the test and the
release never runs. The rest of the handler - the lock bytes `town+0x83C`..`+0x83F` and the
tavern's captains and pirates through the auto-trader chain - is reached before this search
and works, which is why captains do not leak the same way.

## Observed
One save, Reval's tavern, reading the lock variable of the "Fugitive" offer (letter 264, a
pool index far above the 37 merchants) before and after each operation:

|Action|Operation|Lock after|
|-|-|-|
|entering the tavern|type 255, merchant 37 (invalid)|`0xFFFFFFFF` - 255 is past the jump table|
|opening the side room|type 9, merchant 36|**`0x24`** - locked|
|clicking another page|type 9, merchant 37 (invalid)|`0xFFFFFFFF` - released|
|entering that page|type 4, merchant 36|unchanged - type 4 has no handler|
|opening the side room again|type 9, merchant 36|**`0x24`** - locked|
|right-clicking the window closed|type 10, merchant 36|**`0x24`** - not released|

## Fix
Not fixed. The two comparisons above would have to read the letter pool size at
`0x006DD736` instead of the merchant count at `0x006DE4AA` - a four-byte change to each
instruction's operand, leaving the rest of the handler alone.
