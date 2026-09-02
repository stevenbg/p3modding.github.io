# Dismiss Sailors

Opcode `0x05`, handler `0x00537DD0` (thiscall on the operation record). The counterpart
of [hire sailors](./0004-hire-sailors.md), with the same record layout:

```c
struct operation_dismiss_sailors
{
  int field_0_opcode;      // 0x05
  int field_4_ship_index;
  int field_8_count;
};
```

Guards: the ship must have crew (`ship+0x40 > 0`) and be in port (`ship+0x134 < 4`);
the town is resolved from `ship+0x39` and bounds-checked.

The dismissed sailors do not vanish - they **rejoin the town**: both the citizen count
(`town+0x2D4`) and the beggar pool (`town+0x2E4`) rise by the dismissed count. Two
paths:

- `count >= crew`: everyone leaves. Crew and morale (`ship+0x3E`) are zeroed.
- `count < crew`: the crew shrinks and morale drops by `count * 2560 / (crew + 1)`
  (crew as it was before the dismissal), floored at 0.

Both paths recompute the ship's cached cargo figures (`0x005182B0`) and refresh
(`0x00517770`).
