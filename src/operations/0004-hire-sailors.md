# Hire Sailors

Opcode `0x04`, handler `0x00537C20` (thiscall on the operation record), dispatched from
the operation switch's case at `0x0053580C`. What the tavern's Sailors page sends when
sailors are hired onto a ship.

```c
struct operation_hire_sailors
{
  int field_0_opcode;      // 0x04
  int field_4_ship_index;
  int field_8_count;       // the request; the handler clamps it, see below
};
```

The handler resolves the town from the ship's own `+0x39` and clamps the request twice,
so a generous count simply hires "as many as possible":

1. to what the ship still wants - `0x005184F0`, the same routine the tavern UI clamps
   its input field with (see [Crew](../ships/crew.md));
2. to the owner's available sailors in the town - `0x004F6CA0`, the merchant's sailor
   pool capped by the town's beggars (see [Sailor Pools](../merchants/sailors.md)).

Then it applies: the clamped count is drawn from the merchant's sailor pool
(`0x004F6BD0`), added to the crew word `ship+0x40` (a ship whose crew was 0 also gets
its `+0x3F` state and the `0x02` bit of `+0x3D` set), the ship's cached figures are
recomputed (`0x00517770`), the convoy - if any - is marked dirty, and the handler tail
balances the hire against the town's citizen (`+0x2D4`) and beggar (`+0x2E4`) counts.

Guards: the ship index is bounds-checked against the ship count (`0x006DD894`) and the
town against the town count (`0x006DE4B0`); an out-of-range owner (an ownerless ship)
skips the pool bookkeeping.

Note the ship index is read as a full dword - a request built by a mod must not carry
garbage in the upper bits.
