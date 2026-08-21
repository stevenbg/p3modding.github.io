# Sailor Reputation and Sailor Pools

## Sailor Reputation
A merchant's sailor reputation is stored at `0x1f`.
It influences the growth or decline of the sailor pools.
It ranges from `0` to `20` (inclusive).
During the merchant's tick the sailor reputation is increased by `1`, up to a maximum of `20`.
Dismissing a captain sets the sailor reputation to `0`.


## Sailor Pools
The merchant struct's `sailor_pools` array at offset `0xf0` contains an `u8` for every town (indexed by the town's index), which denotes the size of the sailor pool of the merchant in that town.



## Sailors Available for Hire
The pool byte is not the number a merchant can hire. `0x004F6CA0` (thiscall, one
argument, the town index) computes that:

```
cap = [town + 0x2E4] - 1
if cap < 1 { return 0 }
return min(merchant->sailor_pools[town_index], cap)
```

The tavern's sailors page calls it at `0x005D4CB1` for the player merchant
(`[0x006DFC14]`) and its own town index (`window + 0x1BFC`), then caps what it offers at
`50` - the immediate at `0x005D4CD6` that
[mod-tavern-show-all-sailors](./patches/tavern-show-all-sailors.md) raises to `100`.
The cap the getter itself applies, from the town's `+0x2E4`, is a separate limit and
is not affected by that patch.
