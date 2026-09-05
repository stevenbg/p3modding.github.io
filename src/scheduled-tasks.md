# Scheduled Tasks
P3 has a task queue for actions that shall be executed at a given tick in the future.

## Scheduled Tasks Struct
```
00000000 struct scheduled_tasks // sizeof=0x14
00000000 {
00000000     scheduled_task *field_0_tasks;
00000004     int field_4_is_in_use;
00000008     unsigned __int16 field_8_earliest_scheduled_task_index;
0000000A     __int16 field_A;
0000000C     unsigned __int16 field_C_tasks_size;
0000000E     __int16 field_E;
00000010     int field_10;
00000014 };
```
The static scheduled tasks object is at `0x006DD73C`.
The `handle_scheduled_tasks_tick` function at `0x004D85C0` executes all tasks that are due.
It is called at least once per tick.

## Scheduled Task Struct
```
00000000 struct scheduled_task // sizeof=0x18
00000000 {
00000000     unsigned int field_0_due_timestamp;
00000004     unsigned __int16 field_4_next_task_index;
00000006     scheduled_task_opcode field_6_opcode;
00000008     scheduled_task_union field_8_data;
00000018 };
```
The scheduled task's opcdode field denotes which kind of task it is.
Some task types are recurring, and reschedule themselves immediately when they are executed.
The data field is a union containing all possible task arguments.

## The Task Table
The dispatcher switches on the opcode through the jump table at `0x004D8A28`, which
covers opcodes `0x00`..`0x39` (`0x004D8600`: `cmp edx,0x39` / `ja`, then
`jmp [edx*4+0x004D8A28]`); anything above falls through to the requeue tail, so **there is
no task with an opcode above `0x39`**. Two table entries are shared: `0x23`, `0x24` and
`0x34` all point at `0x004D88C9`, and `0x0a` and `0x36` both point at `0x004D872C`.

Most slots are a short stub that calls one handler function, which is what the table below
names; a few do their work inline in the dispatcher instead.

|Opcode|Handler|Task|
|-|-|-|
|`0x00`|`0x004DB330`||
|`0x01`|`0x004DB4E0`|Debt Repayment|
|`0x02`|`0x004DBB00`|[Mayor Election](./scheduled-tasks/0002-mayor-election.md)|
|`0x03`|`0x004DDA40`|[Ten-Day Update](./scheduled-tasks/0003-ten-day-update.md)|
|`0x04`|`0x004DFC94`||
|`0x05`|`0x004E5B84`|[Crime Investigation Result](./scheduled-tasks/0005-criminal-investigation.md)|
|`0x06`|`0x004E2144`|[Update Shipyard Experience](./scheduled-tasks/0006-update-shipyard-experience.md)|
|`0x07`|`0x004E23A4`|[Celebration](./scheduled-tasks/0007-celebration.md)|
|`0x08`|`0x004E2634`|Captain and Pirate Spawning ([Auto Traders](./auto-traders.md))|
|`0x09`|`0x004E2824`||
|`0x0a`|`0x004E2CD4`||
|`0x0b`|`0x004E37F4`||
|`0x0c`|`0x004E38B4`|Land Transport Arrival|
|`0x0d`|`0x004E4984`|Daily Weather ([Port Freezing](./towns/port-freezing.md))|
|`0x0e`|`0x004E4A44`||
|`0x0f`|`0x004E5664`||
|`0x10`|`0x004E63E4`||
|`0x11`|`0x004E6744`||
|`0x12`|`0x004E6A54`||
|`0x13`|`0x004E7A14`||
|`0x14`|`0x004E7BB4`||
|`0x15`|`0x004E7C74`|Marriage|
|`0x16`|`0x004E8234`||
|`0x17`|`0x004E8684`||
|`0x18`|`0x004E8804`||
|`0x19`|`0x004E8E24`||
|`0x1a`|`0x004F6C10`|[Update Sailor Pools](./scheduled-tasks/001a-update-sailor-pools.md)|
|`0x1b`|`0x004ECF64`|Letter and Mission Scripts ([Letters](./letters.md))|
|`0x1c`|`0x004E9094`||
|`0x1d`|`0x004E9564`||
|`0x1e`|`0x004DFD44`||
|`0x1f`|`0x004E0334`|Pirate bands ([Pirates](./pirates.md))|
|`0x20`|`0x004DBBD0`|Writes [`LastWon.eld`](./file-formats/eld.md)|
|`0x21`|`0x004DC1F0`||
|`0x22`|`0x004DB7C0`||
|`0x23`|`0x004E09D4`|Pirate bands ([Pirates](./pirates.md))|
|`0x24`|`0x004E09D4`|Pirate bands ([Pirates](./pirates.md))|
|`0x25`|`0x004E1B84`|Pirate bands ([Pirates](./pirates.md))|
|`0x26`|`0x004E1E54`||
|`0x27`|`0x004DDC00`|Take a retiring captain off his ship ([Auto Traders](./auto-traders.md))|
|`0x28`|`0x004DDF40`||
|`0x29`|`0x004DDE70`||
|`0x2a`|`0x004DE4F0`||
|`0x2b`|`0x004DE790`||
|`0x2c`|`0x004DEA20`, `0x004DEBA0`|both conditional, see below|
|`0x2d`|`0x004E2BC4`||
|`0x2e`|`0x004E9A94`|[Council Meeting](./scheduled-tasks/002e-council-meeting.md)|
|`0x2f`|`0x004EA0D4`||
|`0x30`|`0x004EA594`|Pirate bands ([Pirates](./pirates.md))|
|`0x31`|`0x004EAB94`||
|`0x32`|`0x004EAFE4`|Pirate bands ([Pirates](./pirates.md))|
|`0x33`|`0x004EC934`||
|`0x34`|`0x004E09D4`|Pirate bands ([Pirates](./pirates.md))|
|`0x35`|`0x004E94A4`|[Unfreeze Harbor](./scheduled-tasks/0035-unfreeze-port.md)|
|`0x36`|`0x004E2CD4`||
|`0x37`|inline|Clear a Merchant Flag Bit|
|`0x38`|`0x004ECB14`||
|`0x39`|`0x004ECDF4`||

### The Two Conditional Handlers of `0x2c`
Opcode `0x2c` is the one slot that gates its handlers rather than simply calling one. Its
stub at `0x004D8937`:

- calls `0x004DEA20` only when the date serial `[0x006DE4B4]` minus the task's `data+0x8` has
  reached `0x5D00` ticks (93 days), then restamps `data+0x8` with the current serial - so
  that handler runs at most once a quarter;
- calls `0x004DEBA0` only when the dword at `0x006DF31C` is zero (the dispatcher zeroes `ebp`
  at `0x004D85EB` and compares against it) - which is **always**, since that global is never
  written: see [a global that is only ever read](#a-global-that-is-only-ever-read);
- then reschedules with `due += data+0x4`.

### A Global That Is Only Ever Read
`0x006DF31C` is tested for zero in about **68 places** - the task dispatcher above, the town
tick, `execute_operations`, the captain scan's player branch, and a long tail of UI code -
and **nothing ever writes it**. A byte scan of the executable finds every reference preceded
by a load (`a1`, `8b 0d`, `8b 15`) or a compare (`39`, `3b`): no store, no `lea`, and the
address is never pushed, so no memset or buffer write can reach it either. It sits above
`0x006CC000`, in uninitialised data, so it is zero from process start and stays zero.

Every branch guarded by it therefore always takes the zero path. The sites are all places a
network client would want to skip, which makes "am I a client" the natural reading, but it
cannot be confirmed from this binary and it does not matter for modding: the alternative
path is dead code.

### Clear a Merchant Flag Bit (`0x37`)
Opcode `0x37` has no handler function of its own - the dispatcher does the whole job inline
at `0x004D89D2`. It reads a merchant index from `data+0x4` and a bit number from `data+0x0`,
resolves the merchant record through the accessor `0x005303C0`, and clears that one bit in
the dword at `merchant + 0x608`:

```
edx = 1 << data[0]
merchant[0x608] &= ~edx
```

So it is a deferred "switch this flag off again" task. What the bits of `merchant + 0x608`
mean has not been identified.

## Related Functions
|Address|Function|Description|
|-|-|-|
|0x004D8DD0|reschedule_first_task|Moves the first element to its appropriate position in the queue.|
