# Plague and Fire

Plague and fire are not scripted events. Each is **rolled for one random town at a time**, and
the odds depend on what the town has built: hospitals, chapels and paved streets against plague,
wells against fire. A mission script can also force either outright, which is how the campaigns
use them, but the ordinary source is the roll.

## The Two Rollers

Two sibling functions with the same shape:

| | plague | fire |
|-|-|-|
|roller|`0x004DCA90`|`0x004DCBF0`|
|"already active" flag|`flags & 0x8` (`0x004DCACF`)|`flags & 0x8000` (`0x004DCC30`)|
|operation it enqueues|`0x7C` (`0x004DCBD9`)|`0x7D` (`0x004DCD68`)|
|[scheduled task](../scheduled-tasks.md) that operation creates|`0x1C`, handler `0x004E9094`|`0x1D`, handler `0x004E9564`|

Each call picks its victim with `rand() % [0x006DE4B0]`, the town count, and returns doing
nothing if that town already carries the flag - so a town cannot catch a second plague while one
is running.

Note the roll is **per call, for one town**, not per town per day. What schedules the rollers is
not established, so the percentages below cannot be turned into a per-town rate without measuring
the cadence.

## The Plague Roll

```c
risk = 0;
if (town->field_790_streets_paved * 100 < town->field_792_streets_total * 75)  risk++;
if ((town->field_773_hospitals + town->field_771_chapels) * 50000
        < town->field_2D4_citizens * 8)                                       risk++;
if (rand() % 100 >= 2 * risk + 1) return;                  // no plague
```

**1% at risk 0, 3% at risk 1, 5% at risk 2.** The threshold is the `lea eax,[edi+edi*1+0x1]` at
`0x004DCB49`; the modulus is the `0x64` at `0x004DCB42`. The two multiplications are built from
`lea` chains rather than literals - `*100` at `0x004DCAE6`..`0x004DCAF5`, `*75` at
`0x004DCAF8`..`0x004DCAFE`, `*50000` as `*3125` then `shl 4` at `0x004DCB1C`..`0x004DCB31` - so
searching for the constants finds nothing.

In plain terms, each of the two terms is a condition to **avoid**:

- **fewer than 75% of the town's street tiles are the better grade**;
- **`hospitals + chapels < citizens / 6250`**. One of either covers a town up to 6,250 citizens;
  20,000 citizens needs four.

A chapel counts exactly as much as a hospital here.

## The Fire Roll

```c
risk = (town->field_789_wells * 5000 < town->field_2D4_citizens * 8) ? 5 : 1;
if (rand() % 100 >= risk) return;                          // no fire
```

**1% with enough wells, 5% without** - the `lea eax,[eax*4+0x1]` at `0x004DCC6A`. The threshold is
`wells >= citizens / 625`, four times as demanding as the plague's building term: a town of 3,100
needs **five** wells.

The same flag also picks the severity divisor, `0x1B4E81B5 sar 5` against
`0x51EB851F sar 8` (`0x004DCC93`/`0x004DCCA6`), so wells reduce how big a fire is as well as how
often one starts.

**Wells do nothing against plague**, and hospitals and chapels do nothing against fire. The two
rolls read disjoint fields.

## Where the Counts Come From

`0x00520BE0` is a **town-structure census**. It zeroes the counters, then walks the two completed
structure chains (`town + 0x762` and `town + 0x764`, site records at `town + 0x75C`, building id
at `site + 0x3`, next index at `site + 0x4`) and counts by building id:

|Field|Counts|Building id|Set at|
|-|-|-|-|
|`town + 0x770` / `+ 0x771`|**Chapels** - all / second chain only|`0x2C`|`0x00520C61`, `0x00520D0A`|
|`town + 0x772` / `+ 0x773`|**Hospitals** - all / second chain only|`0x29`|`0x00520C69`, `0x00520CEC`|
|`town + 0x789`|**Wells**|`0x28`|`0x00520C71`|

The plague roll reads the **second-chain** counts `+0x771` and `+0x773`; the fire roll reads
`+0x789`. Building ids are the ones in [Buildings](../reference/buildings.md).

`town + 0x790` and `+ 0x792` are filled by `0x004FA6F0`, which scans the town's tile grid at
`town + 0x7A4` (`[+0x4] * [+0x8]` tiles, array at `[+0x68]`) and counts street tiles:

|Field|Tile bytes counted|
|-|-|
|`town + 0x792`|`0x80`..`0x83` - every street tile (`0x004FA719`, `0x004FA71E`)|
|`town + 0x790`|`0x80` or `0x81` only - the better grade (`0x004FA728`, `0x004FA72D`)|

That scan runs for **one town every sixteen ticks**, gated by
`(town_index & 0xF) == ([0x006DE4A4] & 0xF)` at `0x0051BBF7`, so newly laid street does not
count immediately.

Which byte value is the paved grade rather than the dirt one is inferred from the direction of
the ratio test, not read off the routine that paints tiles.

## While the Flag Is Up

The plague's own handler, task `0x1C` at `0x004E9094`:

- **kills `citizens / 200`** (`0x004E9117`), paid out about four at a time from the poor
  (`town + 0x2E0`) and beggars (`+ 0x2E4`), spread over several ticks and touching only the
  buildings whose id matches the current phase modulo 4;
- **blocks beggar growth entirely** (`0x0051C16D`) - see [Population](./population.md) - so the
  town cannot grow at all;
- **blocks the feed-the-poor influx**, which tests `flags & 0x800008 == 0x800000` and so gets
  nothing while the plague bit is set, without even consuming its trigger bit;
- costs **10 satisfaction** - see [Satisfaction](./population/satisfaction.md);
- shifts the ware price [thresholds](./ware-prices/thresholds.md).

**There is nothing the player can do to end it.** The handler accumulates a counter and clears the
flag itself at `0x004E9486` when that counter passes a limit carried in the task record. It reads
the flag word, the town id, total citizens, the four class counts, `+0x784` and the facility
array - and **never** the structure counters or the
[built-structures mask](../reference/buildings.md#the-built-structures-mask). No building put up
during an outbreak shortens it or softens it.

## The Scripted Route

A [mission script](../letters/mission-scripts.md) can start either disaster directly: its opcode `0xEC`
(handler `0x004EE7D0`) refuses if the town already carries flag `0x8`, then builds the task
payload itself - four severity bytes of `0x31`..`0x50` from `rand()`, a duration term, and terms
scaled by town size - and enqueues operation `0x7C`. Campaign 3 uses it to plague London before
asking the player to rebuild it.

There is also a **dead** disaster starter at `0x0053F810`, which would have created task `0x1C`
at `0x0053F851` and task `0x1D` at `0x0053F88D`. Nothing in the executable references it - no
call, no jump, no pointer - so it never runs.
