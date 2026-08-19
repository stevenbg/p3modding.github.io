# Auto Traders
Both captains and administrators are represented by the same struct.
The following fields have been identified:
```c
struct auto_trader
{
  unsigned __int16 field_0_next_auto_trader_index;
  signed __int16 field_2;
  int field_4_timestamp;
  unsigned __int8 field_8;
  unsigned __int8 field_9_navigation_skill;
  unsigned __int8 field_A_trade_skill;
  unsigned __int8 field_B_combat_skill;
  __int16 field_C_daily_wage;
  char field_E;
  unsigned __int8 field_F_merchant_index;
};
```

## Buying Discount
Auto traders buy cheaper as their trade skill grows. The captain (`0x004D5347`) and
administrator (`0x004FF7E8`) buying routines both compute the percentage of the
transaction price to pay from the auto trader's `field_A_trade_skill`:

```
percent_paid = 2 * (50 - trade_skill / 43)
```

`trade_skill / 43` is the displayed 0-5 skill level, so each level is worth 2%, up to
a 10% discount at level 5 (skill byte 215). The administrator routine applies it right
after `get_buy_price` (`0x004FF944`: `price * percent / 100`, with the operand order
flipped above `0x1000000` to avoid overflowing); its sell orders are settled through
`get_sell_price` without any skill adjustment, so the discount is buying-only. An
office whose administrator index (`office+0x2F2`) is invalid pays 100%.

Office administrators do gain skill like captains do (verified in-game: a long-running
save showed administrator trade levels 1-5), even though the game never displays it -
a level 5 administrator quietly buys everything 10% cheaper.
