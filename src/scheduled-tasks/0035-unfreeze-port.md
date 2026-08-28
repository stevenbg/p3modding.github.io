# Unfreeze Port
The `st_unfreeze_port` function is at `0x004E94A4`. It reopens one iced-over harbour:
it clears bit `0x04000000` of the town flags at `town + 0x2C8` and posts
*"The port of %s is open again."* on the event ticker.

```
00000000 struct scheduled_task_unfreeze_port // sizeof=0x4
00000000 {
00000000     signed __int32 field_0_town_index;
00000004 };
```

The handler takes no arguments. It finds its own record the way any task handler can -
the scheduled-tasks singleton keeps the array base at `+0x0` and the index of the task
being executed at `+0x8` - then reads the town index out of that record's data and
bounds-checks it against the town count at `[0x006DE4B0]` before touching anything.

One task is scheduled per freeze, `level + 1` days out, by the daily ice pass; see
[Port Freezing](../towns/port-freezing.md) for the model that creates them and for the
flag's other readers.
