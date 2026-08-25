# Excess Consumption
The `town_excess_consumption` function at `0x00528630` reduces excess wares which cannot be produced in the town.

Three conditions gate it, and all three must hold:

- the town is not under siege - `test BYTE PTR [esi+0x2c8],0x10 / jne` at `0x0051BD38`;
- `town + 0x76C` is **greater than `0x200`** - `cmp DWORD PTR [esi+0x76c],0x200 / jbe` at
  `0x0051BD41`;
- at least one of `town + 0x2C8` bits `0x20`, `0x40`, `0x80` is set - the routine's own first
  test, `test al,0xe0 / je` at `0x00528639`, which returns without doing anything.

What those three low flags mean is not identified, so how often the routine runs at all is an
open question.

## Calculation
```python
def calculate_excess_consumption(
    daily_production: int,
    daily_consumption_citizens: int,
    daily_consumption_businesses: int,
    current_amount: int,
    t1: int,
    t3: int,
) -> int:
    if daily_production:
        return 0
    if current_amount <= t1:
        return 0

    above_t1 = current_amount - t1
    consumption = daily_consumption_citizens + daily_consumption_businesses
    removed_amount = above_t1
        * 16
        * consumption
        // 10
        // t3

    return min(removed_amount, above_t1)
```
