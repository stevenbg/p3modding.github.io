# Reputation
The function `update_merchant_reputation_and_value` at `0x004F7BB0` calculates the company value and reputation of a given merchant.

Reputation is calculated as follows:
```
reputation = max(0,
    outrigger_rep
    + tenants_rep
    + employment_rep
    + capacity_rep
    + company_value_rep
    + spouse_rep # only in hometown
    + local_social_rep
    + local_trading_rep
    + local_buildings_rep
)
```

## Outrigger
If the merchant is providing the town's outrigger, `outrigger_rep` is set to `1`.

## Tenants
The reputation achieved through tenants is calculated as follows:
```python
def get_tenant_reputation(population_type):
    rent_factor = rent_reputation_factors[rents[population_type]]
    return tenants[population_type] * rent_factor * 0.003

tenants_rep = get_tenant_reputation(rich)
    + get_tenant_reputation(wealthy)
    + get_tenant_reputation(poor)
```

The `rent_reputation_factors` table is at `0x00672DF0`:

|Rent|Reputation Factor|
|-|-|
|None|1.0|
|Low|0.62|
|Normal|0.42|
|High|0.23|
|Very High|0.0|

## Employment
TODO

## Capacity
The merchant's cargo capacity reputation is calculated as follows:
```
capacity_rep = min(5.0, capacity / 100_000.0)
```

## Company Value
The merchant's company value reputation is calculated as follows:
```
company_value_rep = min(5.0, company_value / 100_000.0)
```

## Spouse
A married merchant adds a fixed bonus to his reputation **in his own hometown**, straight from
`field_32_spouse_reputation_bonus` (`merchant + 0x32`) and unscaled:

```c
if (merchant[0x31] < town_count)                      // 0x31 = spouse's hometown, 0xFF = unmarried
    merchant_rep[merchant[0x19]] += (float)merchant[0x32];
```

at `0x004F8198`. The value is the spouse's tier, so it is **0, 1 or 2** - `ernst` contributes
nothing, `normal` one point, `nett` two. An AI merchant can carry 3, because that path rolls the
value rather than deriving it from a tier. Which spouse a merchant is offered, and everything
else the wedding sets, is in [Marriage](./marriage.md).

## Social
`local_social_rep` is the merchant's social reputation in the town.
It is changed through many actions, and degrades over time.

### Recurring Constants
The following values appear in multiple calculations. Both look constant in play, and
neither actually is:

|Name|Measures as|Actually|
|-|-|-|
|base_rep_factor|1.0|`merchant + 0x464`, recomputed per AI merchant by `0x004F42B0` (from scheduled task `0x04`) as `clamp(player_rank / own_rank, 0.7, 1.2)` - a **reputation rubber-band**: rivals trailing the player gain up to 1.2x, a leading rival 0.7x. It measures 1.0 for the player because the player is the reference. Only merchants carrying control-word flag `0x8` - the `difficulty+1` "climbers" marked at world generation (`0x00532420`) - are recomputed at all; everyone else keeps the 1.0 world generation writes. In multiplayer both clamp bounds shift up by `0.05 * difficulty`|
|church_factor|0.0|the difficulty rank at `[0x006DE52C]`, `0..4` - see [Game Settings](../reference/game-settings.md#difficulty). **Zero in every single-player game**: the only writer is operation `0xA9`, whose producer is gated on the multiplayer flag|

### Loans
When granting a loan, `local_social_rep` is increased as follows:
```
local_social_rep += amount / 80_000.0 * interest_factor * base_rep_factor
```
`interest_factor` depends on the chosen interest rate:

|Interest|Factor|
|-|-|
|Very Low|4.0|
|Low|3.0|
|Normal|2.0|
|High|1.0|
|Very High|0.0|

### Church Donations
Donations to the church influence the local social reputation as follows:
```
money_capacity = 12_000 * (church_factor + 1)
effective_amount = min(amount, church_money_capacity - church_money)
local_social_rep += effective_amount
    * 0.0003
    / (church_factor + 1)
    * base_rep_factor
```

Only the reputation uses `effective_amount`: the handler (`0x004FE2D0`) deducts the
**whole donation** from the merchant before the cap is even computed
(`0x004FE2EE`/`0x004FE2F4`), so gold past the cap is simply lost. What the money buys -
the church's decoration level and its decay - is on the [Church](../towns/church.md)
page.

### Church Extension Donation
Donations to the church extension influence the local social reputation as follows:
```
effective_amount = min(amount, church_extension_cost - church_extension_money)
local_social_rep += effective_amount
    * 0.0003
    / (church_factor + 1)
    * base_rep_factor
```

The same caveat as above: `0x004FE420` takes the money first and clamps the fund after,
so overshooting the stage cost wastes the difference. Stage costs, the materials an
extension consumes and its cooldown are on the [Church](../towns/church.md) page.

### Feeding the Poor
The `handle_feeding_the_poor` function is at `0x004FE557` - a method on the town's church
object at `town + 0x794`, reached by operation `0x30` (built by the donation dialog at
`0x005CB0C0`, dispatched inline at `0x00535CF0`). The operation carries the merchant, the
town, a **gate byte**, and the five donation amounts as `u16`, in the order of the
donatable-goods table at `0x006734CC`: grain, beer, fish, meat, wine.

The handler converts each amount to raw units (x2000 for a load ware, x200 for a barrel
ware, by the scale table at `0x00672C14`), takes `min(requested, office stock)` out of the
office - rounding a stock-limited row **down to whole displayed units**, so the fractional
part of the warehouse's last unit is never donatable - and credits the reputation from the
market value of what was actually delivered:

```
for amount, ware_id in delivered:
    local_social_rep += get_sell_price(ware_id, town_index, amount)
        * 0.0003
        / (church_factor + 1)
        * base_rep_factor
```

**The gate byte decides the reply and the side effect.** The dialog computes it as the
donation's market value (each row priced with
[get_sell_price](../towns/ware-prices/selling-price.md) at `0x0052E1D0`) divided by the
town's beggar target:

```
divisor = trunc(sqrt(citizens * poor_satisfaction / 18)) + 8
gate    = min(total_value / divisor, 255)
```

A non-positive product never reaches `fsqrt` (`0x0063AB05` branches away on the sign bit),
so the term contributes nothing and the divisor floors at 8 - a town with unhappy poor is
the **cheapest** to impress, and the threshold scales up with a town's size and
contentment. The three bands, confirmed in game to the single barrel:

|gate|reply|effect|
|-|-|-|
|`< 10`|"...will be grateful to you for your donations."|reputation only|
|`10..49`|"...thank you very much for the generous donation..."|reputation only|
|`>= 50`|"An extremely generous donation! Beggars from everywhere will come to the town..."|reputation, plus bit `0x800000` of the town flags at `0x004FE85A` - the one-shot **beggar influx** trigger, which the beggar code clears when it acts on it (see [Beggars](../towns/population.md#beggars))|

So a large donation always raises the beggar count along with the reputation - the third
reply says so in as many words. Measured example: Luebeck at 3027 citizens and poor
satisfaction 10 has divisor 49, and reaches the third band at exactly 53 barrels of beer.

The dialog's wine row is priced as **salt** when this gate is computed - see
[the bug](../bugs/feeding-the-poor-wine-price.md); the reputation credit is unaffected,
since the handler prices the delivered goods itself.

### Town Coffers Access

### Celebrations

### Crime
The `handle_crime_social_reputation_impact` function at `0x004F8F10` reduces the merchant's social reputation according to the following table:

|Crime Type|Impact|
|-|-|
|0x2|-1.0|
|0x0|-2.0|
|0x9|-2.0|
|0xa|-2.0|
|0xd|-2.0|
|0x1|-4.0|
|0xb|-4.0|
|0x3|-6.0|
|0x4|-6.0|
|0xc|-6.0|
|0xe|-6.0|
|0xf|-8.0|

### Bath House Bribes
A successful bribe increases the social reputation by `min(amount * 0.000099999997, 3.0)`.
Consequently, successfully bribing with 30000 or more gives the maximum social reputation gain.

A failed bribe decreases the social reputation by `2.0`.

### Degradation
The three local components live together in the merchant, **three floats per town at
`merchant + 0x11C`, stride `0xC`**, with the resulting per-town reputation written to
`merchant + 0x2FC + town*4`. The loop at `0x004F81D6` degrades them and sums:

```python
for town in range(town_count):
    social  *= 0.99                  # 0x00672DEC
    trading *= 0.99
    town_reputation = other_contributions + buildings + social + trading
    if town_reputation < 0.0:
        town_reputation = 0.0        # 0x004F8220
```

Slot order is `buildings` (`+0x11C`), `social` (`+0x120`), `trading` (`+0x124`): the
social slot is the one every donation credit targets, e.g. `0x004F8AF0` writing
`12 * (town + 0x18)`. **Only social and trading are multiplied by the `0.99`** - the
buildings term is added untouched, so what a building earns is permanent while everything
under [Social](#social) bleeds away at 1% per update.

## Trading
TODO

## Buildings
`local_buildings_rep` is credited when a **town structure** is added by
[`add_town_building`](../reference/buildings.md#the-built-structures-mask)
(`0x00521900`), in the same block that takes the builder's money. Five of the structures
credit it, and the routine differs per building:

|Building|Id|Routine|Credit|
|-|-|-|-|
|Mint, School|`0x2a`, `0x2b`|`0x004F8C70`|`base_rep_factor`, flat and unconditional|
|Hospital, Chapel|`0x29`, `0x2c`|`0x004F8A50`|`base_rep_factor`, skipped unless a town-population test passes|
|Well|`0x28`|`0x004F89E0`|`base_rep_factor * 0.1` (`0x0066FD5C`), skipped by the ratio in [Well](#well)|

The dispatch is a chain of `cmp` against the building id at `0x00522563`, reached only for
ids above `0x1E`. Every other building id credits nothing. All five write the same slot -
`merchant + 0x11C + town*12` - which is what identifies that slot as the buildings term,
and as [Degradation](#degradation) notes, that slot never decays.

**A merchant has to be behind the build.** `add_town_building` takes the builder's
merchant index as its first argument, and at `0x005223A0` it skips the whole block - the
money *and* the reputation - when that index is at or above the merchant count at
`0x006DE4AA`. A structure raised without a merchant costs nobody anything and credits
nobody.

### Well
The Well is the only one of the five whose credit depends on how many the town already
has, and at a tenth of `base_rep_factor` it is by far the smallest:

```python
wells = town[0x789]                        # count of wells, after the build
if town[0x2D4] < 1:                    return   # 0x004F8A04, also guards the idiv
if wells * 2500 // town[0x2D4] >= 5:   return   # 0x004F8A25
local_buildings_rep += base_rep_factor * 0.1
```

`town + 0x2D4` is the [total citizen count](../towns/population.md), so the test is
`wells * 500 >= citizens` with the multiplication moved across. It reads the count
*after* the new well is included - the Well's case at `0x005220C1` increments
`town + 0x789` before the shared tail reaches the credit.

That matters because the same two numbers gate construction. `0x005220C1` refuses the
build unless `existing_wells * 500 <= citizens`, so the build rule and the credit rule
are near-inverses and **the last well a town will accept always earns nothing**:

|Citizens|Wells the town allows|Of those, wells that pay|Total earned|
|-|-|-|-|
|1000|3|1|0.1|
|3000|7|5|0.5|
|5000|11|9|0.9|

Whether one or two trailing wells earn nothing depends on the citizen count: exactly two
when it is a multiple of 500, one otherwise.
