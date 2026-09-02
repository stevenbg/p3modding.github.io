# Population
A town's inhabitants are counted in four classes, and the first three are not free to grow:
their total is pinned to the number of jobs the town offers. Beggars are the exception and
have a mechanism of their own.

```c
enum population_type : __int32
{
  pt_rich = 0x0,
  pt_wealthy = 0x1,
  pt_poor = 0x2,
  pt_beggar = 0x3,
};
```

|Field|Holds|
|-|-|
|`town + 0x2D4`|total citizens, all four classes|
|`town + 0x2D8`|`i32[4]` the four class counts|
|`town + 0x2E8`|`i32[4]` the previous tick's counts|
|`town + 0x2F8`|`u16[3]` dwelling capacity per class|
|`town + 0x300`|`i16[4]` [satisfaction](./population/satisfaction.md) per class, signed|

Everything below happens in the town's tick, which runs once a day.

## Population Is Four Times the Number of Jobs
Measured across 24 towns at two dates 68 days apart - 48 town-observations - this held
**exactly every time**:

```
rich + wealthy + poor == 4 * jobs
```

where a town's jobs are the posts in its own 21 facilities (`field_4_employees + field_A`)
plus the employees of every merchant building in the town, reached office by office along the
[`office + 0x2CC`](../merchants/trading-office.md) chain.

So **a town grows when its jobs grow**, and by nothing else. The working population cannot
outgrow its employment and cannot fall short of it.

### Nothing Enforces It - the Transfers Simply Preserve It
There is no normaliser. The routine one would expect to find - `0x0051BF90`, which recomputes
the target from the job count and rescales the three classes onto it - is **dead code**: a
byte scan of `.text` finds no call and no jmp to it, and its address appears nowhere in the
executable, so nothing reaches it through a function pointer either.

What the town tick actually runs is `0x0051C650` (levels) -> `0x00520BD0` -> `0x0051C0E0`
(beggars), and it is the beggar routine that recomputes `town + 0x2D4` as the sum of all four
classes, at `0x0051C287` - the one part of the dead routine that survives in live code.

So the identity is not a rule the game enforces but a **consequence of how people move**:
every transfer below shifts exactly four citizens per job, in both directions, and nothing
ever recomputes the total from the job count. A surplus placed into the classes without jobs
behind it is never removed.

The people themselves come **out of the beggar pool**, four at a time, when a facility hires:

### Hiring Converts Beggars Into Citizens, Four To One
`0x005108E0`, reached from the facility employment update, fills a facility's vacancies from
the town's beggars:

```
if poor_satisfaction > 0:
    hired    = min(vacancies, beggars / 4)      # 0x00510910: sar eax,2
    facility.employees += hired                 # 0x00510925
    beggars            -= hired * 4             # 0x00510938
    poor               += hired * 4             # 0x0051093A
```

**Each worker costs four beggars and produces four poor citizens** - the worker and three
dependents - which is exactly why the population is four times the job count. The conversion
is gated on the **poor being satisfied at all**: with `satisfaction[poor] <= 0` the whole
beggar path is skipped and no one is taken on from the street.

So a town with a large beggar pool fills new buildings faster, and a town with none cannot
staff them however many posts it has.

When the beggars run out, the routine falls back to **poaching**: from `0x00510953` it walks
facility slots `1`..`20` - Militia is never drawn from, whichever facility is hiring - and
takes from their `field_A` pools until the vacancies are filled, zeroing a pool only when it
drains one completely. That moves workers between facilities without changing the population
at all.

Note that the job count matching the population is `field_4_employees + field_A`, and
[`field_A`](../reference/facilities.md) is the pool of posts a facility is entitled to but
has not yet filled. So the count is of **posts, filled or not**, and a town's population does
not dip while a facility is part-way through hiring. Measured: `field_A` was non-zero in only
22 of 510 facility slots, and in every one of those the employee count was low - a facility
still filling up.

### What That Leaves for Promotion
Because the three classes only ever gain or lose people together, four at a time, everything
in [Levels](./population/levels.md) is redistribution: promotion and demotion move citizens
between rich, wealthy and poor without changing their total. A tick where satisfaction has
risen looks like `rich +4, wealthy +4, poor -8` - the poor pool being the reservoir the other
two draw from.

### Dwelling Capacity
`town + 0x2F8` holds a capacity per class, and it bounds promotion: the levels calculation
clamps each class's target to it. Across all 24 towns the values share a greatest common
divisor of **20** - observed values run 320, 560, 640, 720, 800, 1120, 1400, 1540, 1680,
1820, 2800, 3360, 3640, 3920 and 4200 - so capacity is granted in units of 20 people.

## Beggars
Beggars sit outside the jobs identity and have their own equilibrium, maintained by
`0x0051C0E0` once per town tick.

### The Target
```
if beggar_satisfaction <= 0:  target = 8
else:                         target = sqrt(total_citizens * satisfaction / 2) / 3 + 8
```

Verified exactly against a live save: the three towns whose beggar satisfaction read `60`
instead of `-20` held 141, 125 and 122 beggars, which is what this formula gives for their
populations of 5329, 4149 and 3942. Every town at `-20` held exactly **8**.

So the driver is beggar satisfaction, and because that value is never lowered while repelling
a siege raises it, a town that fights off attackers permanently raises its beggar equilibrium
- the
[siege beggar satisfaction bug](../bugs/siege-beggar-satisfaction-bonus.md) in its practical
form. It scales with the square root of population times satisfaction, so larger towns attract
proportionally more.

### The Rate
Per town tick, i.e. per day:

```
increase = min(target - current, (sqrt(total_citizens) + 4) / 5)
decrease = min(current - target, (sqrt(total_citizens) + 9) / 10)
```

**Beggars arrive at twice the rate they leave.** For a town of 4442 that is up to 14 a day in
and 7 a day out. Three modifiers apply to the increase:

- `town_flags & 0x8` (`town + 0x2C8`) **blocks it entirely** - the bit is the town's
  active **plague**: scheduled task `0x1C` (handler `0x004E9094`) sets it at
  `0x004E9453` while posting the "Outbreak of the plague in %s" event (type `0x12`)
  and letters to every merchant, and clears it at `0x004E9486` when the outbreak ends.
  Which towns catch one, and what makes it likelier, is in
  [Plague and Fire](./plague-and-fire.md);
- `field_76C & 0x1000` - the town has a [**School**](../reference/buildings.md#the-built-structures-mask) -
  multiplies it by **1.3**, exactly `(13 * increase + 9) / 10` truncated (`0x0051C1B1`).
  That is the School's only effect anywhere in the executable: the bit has three readers,
  this one, the setter, and the AI town planner's "already has one" test at `0x0051F893`.
  Since beggars are the intake for everything else, the School is a growth-rate building -
  it does not move the target, only how fast the town closes on it. Note the multiplier is
  applied **after** the `min` against the deficit and is not re-clamped, so a town with a
  School overshoots its beggar target slightly and settles by oscillating around it;
- if the town has **no militia** (`facility[0].employees == 0`) an increase is floored so the
  result is at least **24**. With a militia there is no floor.

Separately, if `town_flags & 0x800000` is set **and bit `0x8` is clear** - the routine tests
both at once, `and edx,0x800008 / cmp edx,0x800000` at `0x0051C22A` - the flag is **cleared**
and beggars jump at once by `sqrt(total)/6 + target/2`. That jump is capped to keep beggars
below a quarter of the population, but **only when the town already holds more than 50**
(`cmp ecx,0x32 / jle` at `0x0051C25B`); below that the influx is unbounded.

The one thing that sets the flag is a large donation to
[feed the poor](../merchants/reputation.md#feeding-the-poor) (`0x004FE85A`, the only writer
of the bit): a donation whose gate value reaches 50 answers "An extremely generous
donation! Beggars from everywhere will come to the town in order to profit from your
donation" - and this influx is that sentence, executed on the town's next tick.

### What Moves People Into and Out of Beggary
Beggars are the town's intake: every new citizen arrives as a beggar first and is converted by
[being hired](#hiring-converts-beggars-into-citizens-four-to-one). Alongside that continuous
route there are discrete transfers, and every one respects the same four-people-per-job
constant.

|Event|Effect|
|-|-|
|a facility hires (`0x005108E0`)|**4 beggars per worker** become 4 **poor** citizens; needs `satisfaction[poor] > 0`|
|a facility loses workers (`0x0050E380`)|**4 people per lost job** are moved out of rich/wealthy/poor and **into** beggary|
|[form a militia squad](../operations.md) (op `0x41`, `0x0051D4C0`)|militia employees **+5**, beggars **-20**, poor **+20**|
|disband a militia squad (op `0x40`, `0x0051D410`)|militia employees **-5**, beggars **+20**, and 20 taken from the classes **poor first, then wealthy, then rich**, each floored at 1 and the shortfall carried upward (`0x0051D47E`). Needs the militia to hold at least 5|
|[hire sailors](../operations.md) (op `0x04`, `0x00537C20`)|beggars **-N** and total citizens **-N**, one beggar per sailor; if the town has fewer beggars than requested it takes all of them and the pool goes to `0`|
|pay off a ship's crew (op `0x05`, `0x00537DD0`)|the ship's `field_40_crew` is added to beggars and to the total, and the ship's crew is set to `0`|

Note the militia figures: a squad is 5 jobs, and 5 x 4 = 20 people, so it draws exactly the
population those jobs support out of the beggar pool. Sailors are the exception at one beggar
each, because they leave the town with the ship rather than living in it.

Raising a squad has four further conditions, all in the operation `0x41` handler, and it
costs weapons:

- the town's **mayor** (`town + 0x6F1`) must be a **human** merchant (`merchant + 0x8 == 0`,
  `0x0051D580`);
- the pool must hold at least **20 beggars** (`0x0051D54C`) - the twenty the squad takes;
- the town's eight per-type militia counters at `town + 0x998` and `town + 0x99C` must sum
  to **less than the cap** at `town + 0x9A0` (the summing loop at `0x0051D520`);
- **5 weapons** - 50 raw units, the militia wares scaling by 10 - of one of the four
  [militia types](../reference/wares.md) are taken out of the **mayor's own trading office**
  in that town (`office + 0x54 + type*4`, i.e. the office's slots for wares `0x14..0x17`),
  and the squad is refused if that slot holds less (`0x0051D5AF`).

So the militia is armed at the mayor's private expense, one weapon per militia man, and a
town whose mayor is an AI merchant never raises one at all.

Beggars also gate how many sailors a town can offer at all: the
[sailor pool](../scheduled-tasks/001a-update-sailor-pools.md) cap is
`min(100, beggars) * sailor_reputation / 20`.

### The Layoff Split
When `0x0050E380` pushes displaced workers into beggary it does not take them evenly. Each
class gets a weight of `(10 - min(0, satisfaction)) * (citizens - 1)`, so a **dissatisfied
class loses people first**, and a larger class contributes more. Each class keeps at least one
citizen. If more people remain than can be placed, a quarter of the surplus is handed back to
the facility as employees, and the last few are taken one at a time from the poor upward.
