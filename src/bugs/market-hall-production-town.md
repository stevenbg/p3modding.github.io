# Market Hall Production of Town Bug

## Summary
The market hall "Production" page states it displays weekly production, but in the "Town" column it displays the daily production.

## Details
The `ui_prepare_market_hall_window_production_page` function at `0x005DE960` does not multiply the daily production of towns values with `7`, as it does with merchant production.

The town value is read at `0x005DEA3D` (barrel wares) and `0x005DEA98` (bundle wares) from
`town + 0xC4 + ware*4`; the merchant values are accumulated at `0x005DE9CD` from
`office + 0xC4 + ware*4` over the town's office chain and multiplied by 7 at `0x005DE9DB`.
See [Production](../towns/production.md#the-town-and-trader-columns).

## Fix
Replacing the two basic blocks which prepare the market hall page (`0x005DEA18` for barrel wares and `0x005DEA73` for bundle wares) with a copy that does an additional `imul` instruction solves this issue.

![](./market-hall-production-town.png)
