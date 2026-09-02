# Bath House Bribe Success
The `handle_operation_bath_house_bribe_success` function at `0x0053AC50` applies the effects of a successful bribe.
Money is subtracted as expected, and the expenses are added to the monthly expenditure statistic under *"miscellaneous"*.

The merchant's social reputation in the corresponding town is increased as explained in the reputation section.

**There is a check which probably should prevent a merchant from bribing more than 2 councillors in one town. However, this check is bugged, as discussed in the Known Bugs chapter.**

## When a Bribe Succeeds
Success or failure is decided **before** either operation is sent: the bath house
window compares the offer against the councillor's expectation at `0x005B25C3`
(`offer >= expectation` sends 0x42, less sends
[0x43](./0043-bath-house-bribe-failure.md)). The expectation (`0x00529E20`) is:

```
base        = [0, 1, 2, 3, 5, 7, 10, 15][rank]
expectation = (16 + 4*base + rand() % 11) * 500
if the councillor is already bribed by a valid merchant: expectation *= 2
```

`rank` is the merchant's per-town rank byte (`merchant + 0x39C + hometown`, see
[Ranks](../merchants/ranks.md)); its ladder only reaches 5 (Patrician), so table
entries 6 and 7 are unreachable padding - and the switch's default even leaves the
base uninitialized, a latent bug that can never fire. In gold: a rank-0 beginner is
asked 8,000..13,000, a Patrician 22,000..27,000, and buying a councillor **already
bribed by someone else costs double** - which also means another merchant's councillor
can be bought over (the write is unconditional; the last briber wins).

Standing bribes survive until they are consumed by a
[verdict](../scheduled-tasks/0005-criminal-investigation.md#verdict) (two per certain
acquittal), burned by a failed bribe, overwritten by a rival, or wiped world-wide by
the annual [alderman election](../scheduled-tasks/0020-alderman-election.md).
