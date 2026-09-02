# Criminal Investigation

## Begin
When a criminal investigation task is executed for the first time, the status is `crime_investigation_status_pending`.
The task handler sends a *Charge* or *Indictment* letter to the offending merchant, sets the status to `crime_investigation_status_investigating`, and reschedules the task according to the result of the following computation:
```c
(now + (((now & 7) + 8) << 8)) | 0x80
```

The lower 3 bits of the current time are used as a synchronized pseudorandom number ranging from `0` to `7`.
To that number `8` is added, and the result is shifted by `8` to get a timespan between 8 to 15 days.
That timespan is added to the current time, and the 8th bit is set to constrain the time of day between 12:00 and 24:00.

Both random fields are filled with the result of `rand()` with a `RAND_MAX` of `32767`.

## How Investigations Start
Every path in the executable that files this task:

|Creator|Crime|Trigger|
|-|-|-|
|the [tavern pages](../operations/0052-tavern-interaction.md) (`0x53C3B6`/`0x53C4EE`/`0x53C5D8`)|0|talking to the weapons dealer, burglar or pirate - a ~10% roll per page entry|
|operation `0x4F` (`0x0053BA40`)|0|hiring the tavern burglar - the hire also raises the [underworld reputation](../merchants/underworld-reputation.md) and schedules the burglary|
|`0x004E583E`|3 (burglary)|a caught-burglar record; here the task's "random" fields carry case data instead|
|`0x00514FA6`, `0x0054068A`|4 (pirate sponsor)|your hired pirate is caught and confesses|
|`0x004F1754`|8|the mission-script engine - scripted letters can charge you|
|`0x004DCE4A` (via [operation 0x81](../operations/0081-start-criminal-investigation.md))|5..8, random|a periodic task that simply invents charges (indecency, heresy, claiming the world is round, undermining the League) with a chance scaling with the merchant count|
|`0x0050CB80` (helper, called from the sea battle at `0x005FBCB2`)|2, 9..15|**piracy** - see below|

### Piracy: the witness system
A sea battle carries a packed word at `battle + 0x1E94` - `(crime_type << 10) | chance`,
the chance in 1/1000 - and a **witness town** at `+0x1E98`, set at battle creation by
the caller. For a ship attack the witness is the **victim ship's town** (`ship + 0x39`,
pushed at `0x00506FE2`) - a victim with no town means no witness and no charge, ever.

The word starts as crime 2 with chance 0; assaulting a town sets crime 13 at 20%
outright (`0x005FC0B0`). The two halves of the word are maintained separately:

**The crime half is raise-only.** `0x00611649` takes a crime type and writes it only
if it exceeds the current one (`cmp al,bl; jbe skip`), so the word always names the
**worst** deed of the battle; in a town battle the ship crimes 9/11/12 are remapped to
14 first. The deeds:

|Deed|Crime|Site|
|-|-|-|
|your shot hits an enemy ship|9|`0x0060351C`, `0x0061E619`|
|the hit sinks it|11|`0x006035D3`, `0x0061E4F6`|
|plundering a ship|10|`0x0060D228`|
|capturing a ship|12 (14 in a town battle)|`0x0060CD05`, in the ship-takeover routine `0x0060C1C9`|
|firing on a town|14|`0x005FFA49`|
|plundering a town|15|`0x005FEAD2`|

**The chance half is a probabilistic union**: every witness event computes
`P_new = 1 - (1 - P_old) * (1 - p_deed)` in per-mille (the shared math of
`0x00611299`, inlined at every site). The big contributions come at battle end, when
the result screen (`0x0060D299`) walks every enemy ship and unions a weight per fate:

|Enemy ship's fate|Witness weight|
|-|-|
|captured (its side byte `+0x142` changed)|**20%**|
|sunk|**10%**|
|fled, plundered, or simply survived|**5%**|

On top of that, ships watching from the local town's harbor add 5% each
(`1 - 0.95^n`, square-and-multiply at `0x00605A62`), and ships near the engagement on
the scrollmap add 5% each through operation `0x9F` (sent at `0x0050B5E1`, handled at
`0x00536CD5`).

When the battle resolves, `rand % 1000 < chance` (two sites, `0x005FBC8B` and
`0x00603ECA`) files this task about five days out, with the offending ship's index in
the task's `+0x12` - the `%s` of the [indictment](../letters/19-indictment.md)'s "your
ship %s". So capturing a single ship with no other witnesses around is a flat 20%
charge; capture two and it unions to 36%; sink the escort as well and it climbs to
42% - while a victim with no home town can never testify at all.

## Verdict
The second execution (status `investigating`, branch `0x004E5E09`) decides the outcome
with the town's **councillor bribes** - the four bytes at `town + 0x6DC..0x6DF`, one
per bath-house councillor, each holding the index of the merchant who bribed him
(`0xFF` = nobody; see the [bath house bribe](../operations/0042-bath-house-bribe-success.md)).
The loop scans them for the accused's index and **consumes** what it finds (writing
`0xFF` back), stopping after two:

- **two matches**: acquitted - the verdict letter (type `0x4A`) is delivered
  (`0x004E5E8B`, human merchants only);
- **one match**: acquitted iff `random1 <= 0x4E20` - 20000/32767 ≈ **61%**;
- **no match**: guilty.

So a standing pair of bribes buys exactly one certain acquittal, a single bribe a 61%
chance, and an unbribed defendant is always convicted.

## The Fine
A guilty verdict dispatches by crime type (`jmp [crime*4 + 0x4E639C]` at `0x004E5E80`)
to compute the fine's size in units of `random2`:

|Crime types|Units|
|-|-|
|2 (pirate attack), 5, 6, 7 (indecency, heresy, round world)|`random2 % 7 + 2` (2..8)|
|15 (pirate plundering town)|`random2 & 7 + 10` (10..17)|
|0 (criminal plans), 9, 10, 13 (pirate firing/plundering ships, attack on town)|`random2 % 11 + 10` (10..20)|
|1 (boycott broken), 8 (undermining league), 11 (pirate sinking ships)|`random2 % 21 + 30` (30..50)|
|3 (burglary), 4 (pirate sponsor), 12, 14 (pirate capturing ships, firing on town)|`random2 % 21 + 60` (60..80)|

The common tail (`0x004E604F`) then scales by the merchant's wealth:

```
fine = (units * max(company_value, 5000) / 500000 + 1) * 500   gold
```

with the company value read from `merchant + 0x46C` and the whole product capped
through 64-bit arithmetic - **the fine scales linearly with company value**. A
millionaire caught making criminal plans (units 10..20) owes 10,500..20,500 gold; the
same crime at the 5,000-florin floor costs 500..1,000.

## The Reputation Cost

A conviction also costs social reputation. `0x004F8F10(town, crime)` - called only
from the guilty tail (`0x004E6062`/`0x004E607A`), once for the trial town and once for
the merchant's hometown - subtracts a per-crime constant from the town's social
reputation float (`merchant + 0x120 + town*12`; index map `0x004F8FC4`, jump table
`0x004F8FAC`):

|Crime types|Penalty|
|-|-|
|2 (pirate attack)|-1.0|
|0, 9, 10, 13 (criminal plans, firing on/plundering ships, attacking town)|-2.0|
|1, 11 (boycott broken, sinking ships)|-4.0|
|3, 4, 12, 14 (burglary, pirate sponsor, capturing ships, firing on town)|-6.0|
|15 (plundering town)|-8.0|
|5, 6, 7, 8 (the invented charges)|none - these cost only the fine|

The tiers mirror the fine-unit tiers. An acquittal skips both the fine and the
penalty. Note that a conviction does **not** touch the
[underworld reputation](../merchants/underworld-reputation.md) - that is raised by
committing the deeds, not lowered by being caught.

## Scheduled Task Data
The following task fields have been identified:
```c
struct scheduled_task_criminal_investigation
{
  unsigned __int8 field_0_merchant_index __tabform(NODUPS);
  unsigned __int8 field_1_town_index;
  char field_2;
  unsigned __int8 field_3_hometown_index;
  int field_4_timestamp;
  crime_type field_8_crime_type;
  crime_investigation_status field_9_status;
  unsigned __int16 field_A_random1;
  unsigned __int16 field_C_random2;
  signed __int16 field_E;
};
```

where `crime_type` was found to be:
```c
enum crime_type : unsigned __int8
{
  crime_type_criminal_plans = 0x0,
  crime_type_boycott_broken = 0x1,
  crime_type_pirate_attack = 0x2,
  crime_type_burglary = 0x3,
  crime_type_pirate_sponsor = 0x4,
  crime_type_indecent_behaviour = 0x5,
  crime_type_heresy = 0x6,
  crime_type_round_world = 0x7,
  crime_type_undermining_league = 0x8,
  crime_type_pirate_firing_on_ships = 0x9,
  crime_type_pirate_plundering_ships = 0xA,
  crime_type_pirate_sinking_ships = 0xB,
  crime_type_pirate_capturing_ships = 0xC,
  crime_type_pirate_attacking_town = 0xD,
  crime_type_pirate_firing_on_town = 0xE,
  crime_type_pirate_plundering_town = 0xF,
};
```

and `crime_investigation_status` was found to be:
```c
enum crime_investigation_status : unsigned __int8
{
  crime_investigation_status_pending = 0x0,
  crime_investigation_status_investigating = 0x1,
  crime_investigation_status_confiscation_successful = 0x2,
  crime_investigation_status_unknown = 0x3,
};
```
