# Feeding the Poor Prices Wine as Salt

## Summary
The church's "feeding the poor" donation dialog computes the donation's value with the wine
row priced as **salt** - an eighth of wine's base price - so wine barely counts toward the
donation thresholds. The reply bands and the beggar influx are decided by that value, so a
wine donation that should rate "extremely generous" rates "generous" at best; the
reputation actually credited is unaffected.

## Details
The dialog (window code at `0x005CAF7B`-`0x005CAFF3`) walks its five edit controls -
grain, meat, fish, beer, wine in control order - and prices each row with
[get_sell_price](../towns/ware-prices/selling-price.md), summing into the window's running
total at `+0x1D50`, which becomes the donation's gate byte (see
[Feeding the Poor](../merchants/reputation.md#feeding-the-poor)).

The loop uses its row counter directly as the ware id, in both places that need one:

```asm
005caf83  mov  al, [ebp+0x672C14]     ; barrels/loads scale byte, ware = row counter
...
005cafce  push ebp                    ; ware argument of get_sell_price = row counter
005cafcf  mov  ecx, 0x6DE3D8
005cafd4  call 0x0052E1D0
```

In control order the ware ids should be `0, 1, 2, 3, 7` (grain, meat, fish, beer, wine);
the counter yields `0, 1, 2, 3, 4`. The first four coincide, so only the wine row is wrong,
and it is priced as ware 4 - **salt**, base price `0.1425` against wine's `1.1`. The scale
byte survives by coincidence, salt and wine both being barrel wares.

The `handle_feeding_the_poor` handler (`0x004FE557`) indexes wares correctly through the
donatable-goods table at `0x006734CC`, so the reputation credit and the warehouse deduction
never see the wrong price. Only the gate byte does - which decides the reply and whether
the beggar influx fires.

Measured in Luebeck (gate divisor 49): 65 barrels of wine scored gate byte **22**, i.e.
about `0.083` gold per raw unit - 1.16x salt's base and 0.15x wine's, unambiguously salt.
Reaching the "extremely generous" band with wine alone took **162** barrels where correct
pricing needs **7**.

## Fix
`mod-fix-feeding-the-poor-wine-price` hooks the dialog's single `get_sell_price` call
(`0x005CAFD4`, the only call to it in the window's range) and remaps ware 4 to 7 before
calling the original. Salt is never legitimately passed at that site - the dialog's rows
are wares 0, 1, 2, 3 and 7 - so the remap cannot misfire, and no other caller is touched.

Confirmed to the barrel: with the fix, 7 barrels of wine in the same town score gate **51**
(extremely generous, the predicted minimum) and 6 barrels score **44**.
