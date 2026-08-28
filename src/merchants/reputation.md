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
Every spouse has a fixed reputation bonus.
TODO list options

## Social
`local_social_rep` is the merchant's social reputation in the town.
It is changed through many actions, and degrades over time.

### Recurring Constants
The following values appear in multiple calculations, and appear to have fixed values.

|Name|Value|Location|
|-|-|-|
|base_rep_factor|1.0|Merchant|
|church_factor|0.0|GameWorld|

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

### Church Extension Donation
Donations to the church extension influence the local social reputation as follows:
```
effective_amount = min(amount, church_extension_cost - church_extension_money)
local_social_rep += effective_amount
    * 0.0003
    / (church_factor + 1)
    * base_rep_factor
```

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

## Trading
TODO

## Buildings
TODO
