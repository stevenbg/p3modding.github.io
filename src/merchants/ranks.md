# Ranks
Merchants may rank up in their hometown on the first day of every month.
The office "Personal" page gives rough hints whether more wealth or more reputation is needed to get to the next level, until the following wealth and reputation requirements are met:

|Minimum Reputation|Rank|
|-|-|
|5|Trader|
|7.5|Merchant|
|10|Travelling Merchant|
|15|Councillor|
|25|Patrician|

|Minimum Company Value|Rank|
|-|-|
|100,000|Trader|
|200,000|Merchant|
|300,000|Travelling Merchant|
|500,000|Councillor|
|900,000|Patrician|

However, to actually reach the next rank, the following reputation values must be reached:

|Minimum Reputation|Rank|
|-|-|
|7|Trader|
|12|Merchant|
|20|Travelling Merchant|
|40|Councillor|
|60|Patrician|

## Building Permits
Once you reach the Trader rank in a town, it'll grant you the building permit.

## Where the Rank is Stored
A merchant's rank is kept **per town**, as a byte at `merchant + 0x39C + town_index`, and
computed by the code in front of `update_merchant_reputation_and_value` (`0x004F7653`,
`0x004F7699`, `0x004F78B1`, `0x004F79B6`, `0x004F7A27`, `0x004F7A99`, `0x004F7AE2`,
`0x004F7B0E`, `0x004F7B31`) from the per-town reputation float at
`merchant + 0x2FC + town_index*4` and the company value at `merchant + 0x46C` - the
`0xDBBA0` = 900,000 comparison at `0x004F7AD4` is the Patrician step of the table above.
Observed values in a live 24-town game run 3..5 for the AI merchants, and the
[pirate AI](../pirates/ai.md#the-decision-to-attack) reads the home-town entry as its
"is this merchant worth robbing" test - for **player-owned** prey only, so the rank the
player holds at home is what decides whether pirates bother with him at all.
