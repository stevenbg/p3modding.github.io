# Satisfaction
P3's setting "Needs of the citizens" changes how easy it is to increase the satisfaction, and how fast it changes.
The *satisfaction classes* are displayed in-game: *Very happy*, *happy*, *very satisfied*, *satisfied*, *dissatisfied*, and *annoyed*.
The satisfaction for each population type is stored in the town's *satisfactions* array at offset `0x300`, holding a **signed** `i16` for every population type - beggars included, at `+0x306`.
The beggar entry exists but `update_citizen_satisfaction` never maintains it, which is what
makes the [siege beggar satisfaction bug](../../bugs/siege-beggar-satisfaction-bonus.md)
permanent: nothing ever brings it back down. Measured across 24 towns, it reads `-20` in 21
of them and `60` in the three that have repelled sieges.

The three maintained entries are read as signed elsewhere too - the
[construction workforce](../construction.md) target subtracts a weighted sum of them when it
is negative.
The function `prepare_citizens_menu_ui` at `0x0040B570` calculates the satisfaction classes by converting the `i16` into an `f32`, and picking the highest applicable class:

|Satisfaction >|Satisfaction Class|
|-|-|
|29.5|Very Happy|
|19.5|Happy|
|9.5|Very Satisfied|
|-0.5|Satisfied|
|-10.5|Dissatisfied|
|-Infinity|Annoyed|

The function `update_citizen_satisfaction` at `0x0051C830` calculates the *current satisfaction* each population type would have.
The satisfaction is then increased or decreaseed by the respective *step size*, depending on whether it was bigger or smaller than the current satisfaction, but it won't go beneath -40 or above 80.
At `0x006736AC` there is a table that contains for every difficulty the step sizes for increments and decrements to the satisfaction:

|Needs Setting|Increment|Decrement|
|-|-|-|
|Low|3|1|
|Normal|2|1|
|High|1|2|
|Unused|1|1|

At `0x006736A0` there is a table that contains for every difficulty the *base satisfaction* for every population type:

|Needs Setting|Rich|Wealthy|Poor|
|-|-|-|-|
|Low|-7|-12|-20|
|Normal|-13|-18|-27|
|High|-20|-25|-32|

Within `update_citizen_satisfaction` 6 *situational modifiers* are implemented:

|Situation|Impact|
|-|-|
|Siege|-10|
|Pirate Attack|-8|
|Plague|-10|
|Blocked|-6|
|Boycotted|-4|
|Famine|-10|

At `0x00672938` there is a table that defines *ware satisfaction weights* for every population type:

|Ware|Rich|Wealthy|Poor|
|-|-|-|-|
|Grain|2|4|8|
|Meat|5|4|4|
|Fish|2|6|6|
|Beer|2|6|6|
|Salt|2|2|4|
|Honey|3|2|0|
|Spices|3|0|0|
|Wine|5|2|0|
|Cloth|5|4|0|
|Skins|3|2|0|
|WhaleOil|3|4|4|
|Timber|3|4|6|
|IronGoods|2|2|0|
|Leather|2|2|4|
|Wool|2|6|4|
|Pitch|0|0|0|
|PigIron|0|0|0|
|Hemp|0|0|0|
|Pottery|3|2|4|
|Bricks|0|0|0|
|Sword|0|0|0|
|Bow|0|0|0|
|Crossbow|0|0|0|
|Carbine|0|0|0|

The reference quantity is the ware's **t0 [price threshold](../ware-prices/thresholds.md)**
(`town + 0x4F0 + ware*0x10`, read at `0x0051CAF5`), not a consumption figure. t0 is normally
seven days of consumption, which is why it behaves like one - but it is fourteen days under
siege, fire, famine or plague, it carries the building-material bonuses that make timber and
bricks far larger, and it is floored at 1000 or 10000. A ware is skipped entirely unless
`town + 0x310 + ware*4` (the citizens' consumption) is positive (`0x0051CAE4`).

The current satisfaction is calculated as follows:
```
def get_ware_satisfaction(ware_id, population_type):
    if citizen_consumption[ware_id] <= 0:
        return 0                      # 0x0051CAE4
    t0 = thresholds[ware_id][0]
    if wares[ware_id] >= 2 * t0:
        return satisfaction_weights[population_type][ware_id]
    else:
        # truncated toward zero, so the negative branch rounds up, not down
        return trunc((wares[ware_id] - t0)
            * satisfaction_weights[population_type][ware_id]
            / t0)

current_satisfaction = 2 * (
    base_satisfaction
    + situational_modifiers
    + unknown_modifiers # 9 total, 8 capped at 4
    + ware_satisfactions
)
```
