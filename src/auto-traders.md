# Auto Traders
Captains, administrators and pirate captains are represented by the same struct. The array
lives behind the ships container at `0x006DD7A0` (array pointer at `+0x0`, count
word at `+0xF2` = `0x006DD892`, stride `0x10`). Each town chains its auto traders
through `field_0_next_auto_trader_index`, headed by the town's
`field_82E_auto_trader_chain_head` (see [Towns](./towns.md)). The chain mixes two
record kinds, discriminated by `field_8` (check `0x004FE150`, true for
`field_8 > 0x20`). The record initializer (`0x004FDF50`) fills `field_8` with a
random byte and reduces it `% 11` for captains (0..10), so the range encodes the
kind: captains always pass `<= 0x20`, the pirate captains - one record per town,
maintained by the spawn task - fail it. (The pirates are the tavern characters a
ship can be handed to; identified by matching the records' name ids against the
pirate captains in-game.) For pirates `field_8` doubles as the greed byte: the
loot share a tavern pirate demands is `25 + 5 * ceil(field_8 / 32)`, i.e.
35%..65% - verified against seven live pirates. The initializer also rolls
`field_2`/`field_3` as first/last
name ids (modulo the name-registry counts `0x006DDB70`/`0x006DDB74`), splits a
600-point budget randomly across the three skills, and stamps `field_4` from the
current date serial. The two kinds also differ in their wage formula: captain
records derive it from the trade skill alone (`0x004FE160`), pirate records from
the sum of all three skills plus a base. A town's tavern offers a captain
for hire while the chain contains a captain record no merchant employs
(`field_F_merchant_index` = `0xFF`): the captain resolver `0x005269A0`(town,
merchant) walks the chain applying exactly that, preferring a captain the asking
merchant already employs; the sibling resolver `0x005261D0` does the same for
the pirate captains. Verified against a live save: exactly the towns whose
taverns showed captains had a matching chain record.
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

## Array Layout
Observed live (128-record array): indices 0..47 hold one world-generation pair
per town - the founding captain of an AI merchant's starting ship (elite skills,
exceeding the 600-point budget of the normal roller; verified by matching the
records against the AI ships' `field_42_captain_index` and owners) and the
town's pirate. The dynamic range
above holds spawned tavern captains and employed records; the tail is free
capacity, `memset` to `0xFF` and freelist-linked through `field_0` (the
allocator at `0x005097C0` grows the array by 64 records). Dismissing an office
administrator frees his record back to the freelist; re-employing allocates a
fresh one (new name and skills, wage 10, trade skill 0) - whatever the office
remembers about a previous administrator is stored on the office, not in this
array.

A record's situation is encoded by chain membership, verified across live save
states: chained to a town = sitting in that town's tavern; unchained = serving
on a ship (`field_42_captain_index`). The record's merchant byte is only
maintained for the player - AI merchants' hires keep `0xFF` - so ship ownership
comes from the ship's `field_0_merchant_index` (AI merchants hold low indices
with two starting ships each; pirate ships and empty ship slots hold `0xFF`).
Records never expire; the population circulates between taverns and decks.

## Captain and Pirate Spawning
A periodic task (`0x004E2634`, rescheduling itself in date-serial ticks, ~345
per day) maintains both populations. Per town it records two flags: whether the
captain resolver finds an unemployed captain (called **with the merchant count
as the asking merchant** - a value no real merchant has, so only records with
`field_F_merchant_index` = `0xFF` count), and whether the pirate resolver finds
a pirate.

- **Pirates**: when fewer than 3 towns have a pirate, one is spawned into a
  pirate-less town (`0x00526A50(town, 1)` - the pirate initializer path).
- **Captains**: at 8 or more unemployed captains the task just reschedules far
  out (`+0x700` ticks, ~5 days). Below that it compares the count against a
  demand target derived from fleet statistics (`0x00509930` on the ships
  container) and spawns a captain into a captain-less town when the count is at
  or below the target, or below 2 (`0x00526A50(town, 0)`), rescheduling `+0x200`
  ticks (~1.5 days) after a spawn and `+0x400` otherwise.

No expiry logic exists in the task, and `field_4` is never compared against the
current date: an unhired captain stays until somebody hires him - including AI
ships, which fill their `field_42_captain_index` through the same resolver and
unlink the captain from the town (`0x0051A1B9`). A captain "disappearing" from a
tavern is somebody else's hire, not a timeout.

Dismissing a captain re-links him into the town's chain still carrying the
dismissing merchant's index; it only flips to `0xFF` (generally hireable) when
that town's tavern is next opened (observed in-game). Until then the spawn task
does not count him - so dismissing captains without revisiting their taverns
makes the game under-count and spawn extras, pushing the world above the usual
two hireable captains (four observed live).

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
