# Levels
The `town_update_population_levels` function at `0x0051C650` determines how many citizens are promoted and demoted and how many poor citizens emigrate.

This only ever **redistributes**. The three working classes gain and lose people only through
the discrete transfers described in
[Population](../population.md#population-is-four-times-the-number-of-jobs), four at a time per
job, so promotion and demotion move citizens between rich, wealthy and poor without changing
their total - the poor pool being the reservoir the other two draw from.

The following pseudocode denotes the calculation:
```python
class Town:
    citizens: list[int]
    satisfactions: list[int]
    has_mint: bool
    dwellings_capacity: list[int]

    def __init__(
        self,
        rich: int,
        wealthy: int,
        poor: int,
        satifsaction_rich: int,
        satisfaction_wealthy: int,
        satisfaction_poor: int,
        has_mint: bool = False,
        dwellings_capacity_rich: int = 999999,
        dwellings_capacity_wealthy: int = 999999,
        dwellings_capacity_poor: int = 999999,
    ):
        self.citizens = [rich, wealthy, poor]
        self.satisfactions = [
            satifsaction_rich,
            satisfaction_wealthy,
            satisfaction_poor,
        ]
        self.has_mint = has_mint
        self.dwellings_capacity = [
            dwellings_capacity_rich,
            dwellings_capacity_wealthy,
            dwellings_capacity_poor,
        ]

    def update_population_levels(self):
        self.update_population_level(0)
        self.update_population_level(1)
        # TODO poor emigration

    def update_population_level(self, level: int):
        target = (
            self.citizens[2]
            * (self.satisfactions[level] + 40)
            // self.get_divisor(level)
        )
        # The capacity clamp has a floor of its own: 0x0051C6EA takes max(capacity, 10)
        # rather than the capacity itself.
        if self.dwellings_capacity[level] < target:
            target = max(self.dwellings_capacity[level], 10)
        else:
            target = max(target, 1)
        LOGGER.debug(f"{level} target: {target} stock: {self.citizens[level]}")
        if target < self.citizens[level]:
            # Current stock exceeds target
            demoted = 2 * self.citizens[level] // target + 1
            if demoted > self.citizens[level]:
                demoted = self.citizens[level] - 1
            self.citizens[2] += demoted
            self.citizens[level] -= demoted
        else:
            # Target exceeds current stock
            if self.citizens[level]:
                promoted = 2 * target // self.citizens[level] + 1
            else:
                promoted = 1  # Avoid division by zero
            if self.citizens[2] > promoted:
                LOGGER.debug(f"promoting {promoted} poors to {level}")
                self.citizens[2] -= promoted
                self.citizens[level] += promoted

    def get_divisor(self, level):
        if level == 0:
            return 213 if self.has_mint else 320
        elif level == 1:
            return 160
        raise Exception()
```

`has_mint` is **bit `0x400` of the [built-structures
mask](../../reference/buildings.md#the-built-structures-mask)** at `town + 0x76C`, tested
at `0x0051C671`. So a Mint lowers the divisor on the rich target from 320 to 213 - a
**+50% rich-citizen target** for the same poor population and satisfaction. (Both
divisors are reciprocal-multiply constants: `0x66666667 >> 7` is `/320`, and
`0x99D722DB >> 7` with the add-back at `0x0051C67D` is `/213`.)

**That is the Mint's only effect.** The bit has exactly three readers in the executable -
this divisor, the setter, and the AI town planner's "already has one" test at
`0x0051F87B` - so the Mint does nothing to money, interest, taxes or trade despite its
name.

For a fixed number of total inhabitants and satisfactions, the groups converge:
![image](./levels1.png)
