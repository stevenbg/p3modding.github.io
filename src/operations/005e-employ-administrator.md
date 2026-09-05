# Employ or Dismiss Administrator

Opcode `0x5E`, handler `0x0053D990` (thiscall on the operation record). What the trading
office window's hire button sends (`0x005DC945`, from the window's town at `+0xECC8` and
the player merchant `[0x006DFC14]`); mode 1 is the dismissal.

```c
struct operation_administrator
{
  int   field_0_opcode;      // 0x5E
  short field_4_merchant;
  short field_6_unused;
  short field_8_town;
  short field_a_unused;
  int   field_c_mode;        // 0 employ, 1 dismiss; anything else does nothing
};
```

The handler resolves the merchant's office in the town (`0x005308A0(merchant, town, 0)`)
and returns if there is none. Then:

- **mode 0, employ** (`0x0053D9E6`): only if `office+0x2F2` holds no live auto-trader
  index (at or above the count `[0x006DD892]`). It takes a record from the auto-trader
  freelist (`0x005097C0`), stores the index at `office+0x2F2`, zeroes the record's trade
  skill (`+0xA`) and hands it to the administrator wage formula `0x004FE160`. Nothing is
  charged here; an office that already has an administrator is left alone.
- **mode 1, dismiss** (`0x0053D9B0`): only if the office has one. Frees the record back to
  the freelist (`0x005098B0`) and clears bits `0x1` and `0x2` of `office+0x2D6`.

A dismissed administrator is gone for good: the freed record is what the next hire
reuses, at trade `0` - see [Administrators](../auto-traders/administrators.md).
