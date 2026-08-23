# Tavern Missions (type `0x71`)

The missions a tavern's side room offers - patrol, escort, courier, smuggler, trader,
pirate hunter, fugitive, treasure map - are not a structure of their own. Each offer is
a [scripted letter](./scripted-letters.md) of type `0x71` in the recipient's letter
chain, paired with a scheduled task that carries the mission's script variables. The
letters window does not list them.

## Finding a Town's Offers
`0x004D7900` (thiscall(this = the message pool `0x006DD730`, start index, town)) is the
game's own search. It walks the chain from `start index` through each message's `+0x6`
and returns the first index where

- the type (`+0x4`) is `0x71`,
- the town byte (`+0x5`) is the town asked for,
- and the descriptor's `+0x4` date is still in the future.

It returns `0xFFFF` at the end of the chain. The tavern side room starts at the player's
own chain head - `0x005303C0(0x006DE4A0, player merchant)` `+0xA` - and continues each
search from the found letter's `+0x6` (`0x005A7223`, `0x005A72C7`).

## The Descriptor
The letter's `+0x8` descriptor holds:

|Offset|Meaning|
|-|-|
|`+0x0`|length of the letter text|
|`+0x4`|date the offer expires; `0x004D7900` requires it to be in the future|
|`+0x8`|index of the mission's scheduled task (u16)|
|`+0xC`|which of that task's script variables holds the merchant who took the offer|

The letter's `+0xC` text buffer begins with the offer's NUL-terminated **title** -
"Patrol", "Escort", "Treasure map" - which is what the side room draws as its window
title (`0x005D7FF2`); the body follows after that terminator. Escort and fugitive
missions share one script, so the title is what separates the kinds, not a type byte.

## The Mission Task
The task lives in the [scheduled task](../scheduled-tasks.md) pool `0x006DD73C` and
carries opcode `0x1B`; the side room refuses an offer whose task does not
(`0x005A7279`). Running that task runs the mission's letter script: the opcode's handler
enters the interpreter at `0x004ECF64`, which reads the task as

|Field|Meaning|
|-|-|
|`+0x8`|the script blob|
|`+0xC`|the script's variable array|
|`+0x10`|script id|
|`+0x12`|number of variables|
|`+0x14`|program counter|

The blob starts with the command count (u16), the variable count (u16), then one dword
offset per command; a command's bytes are at `blob + offsets[pc]` and its first byte is
the command. So `offsets[0]` is always `4 + count * 4`.

The task's own due date (`+0x0`) is computed at creation as today's date plus one of the
variables (`0x00511527`), which is the deadline of the accepted mission - a different
clock from the offer's expiry in the descriptor.

## What an Offer Is Worth
A command dispatches through the index table at `0x004F3054` into the handler table at
`0x004F2E34`, about 135 handlers for commands `1..0xFB`; the arithmetic the mission data
is built from works on the variable array:

|Command|Bytes|Meaning|Handler|
|-|-|-|-|
|`0A dst imm32`|6|`var[dst] = imm32`|`0x004EEEAA`|
|`0B a b dst`|4|`var[dst] = var[a] + var[b]`|`0x004EEED1`|
|`0C a b dst`|4|`var[dst] = var[a] - var[b]`|`0x004EEEF3`|
|`0D a b dst`|4|`var[dst] = var[a] * var[b]`|`0x004EEF15`|
|`0E a b dst`|4|`var[dst] = var[a] / var[b]`|`0x004EEF36`|
|`09 mod dst`|3|`var[dst] = pseudo-random % var[mod]`|`0x004EEDFD`|
|`F8 merchant amount`|3|pays `var[amount]` to that merchant|`0x004ED20D`|

Every mission's figures can be read from its own [script file](./mission-scripts.md),
which is where this table comes from - `F8` is what proves a variable is money, and the
letter templates name the variables they print (`%c` a sum, `%t` a town, `%B` an amount
of cargo), which confirms each one independently:

|Script|Offer|Cargo|Destination|Money|
|-|-|-|-|-|
|8 smuggler|`var0` = the tavern's town|`var3`, rolled as `rand%40 + 4`|`var5`, a random town re-rolled against `var0`, named only after acceptance|`var3 * 150`, worked out when it pays|
|9 trader|`var0`|`var3`, same roll|`var5`, stated in the offer|`var3 * 90`, worked out when it pays|
|11 pirate hunter|`var0`|-|-|`var5 = (rand%3 + 1) * 1500 + rand%10 * 100`|
|12 treasure map|`var4`|-|-|**costs** `var1 = rand%7 * 100 + 800`; the treasure is `var8`, filled in at the end|
|13 courier|`var0`|3 loads, as literal text|a chain of towns|`(days_to_spare * 220) + 50`, at the end only|
|15 escort, fugitive|`var1`|-|`var10`, from the route command `0x29`|`var13 = rand%20 * 100 + 3000`|
|16 patrol|`var0`|-|a chain of towns|`var4 = (rand%3 + 1) * 1700` per foiled ambush; the payout is `var9`|

Scripts 11, 15 and 16 compute their sum before they send the offer, so a pending offer
already holds it. The two transport orders hold no sum: the rate is fixed in the script
and applied to the cargo at the moment it pays, so their worth has to be derived from the
cargo. The patrol's figure is a rate, not a fee, and its voyage pay is never stated. A
smuggler additionally risks losing the goods, the chance falling the more other cargo his
ship carries.

Variables are reused as a script runs - a delivered order overwrites its answer slot with
the sum it paid - so these meanings only hold for an offer still on the table.

## Who Holds an Offer
The variable named by `descriptor+0xC` holds `0xFFFFFFFF` while nobody has taken the
offer and a merchant index afterwards. The side room accepts the letter when that
variable is the asking merchant or is at or above the merchant count (`0x006DE4AA`), and
walks past it otherwise (`0x005A7291`-`0x005A72A3`). Nothing in the field distinguishes
human players from AI merchants.

The index is a **tavern lock**, of the same kind the tavern's other persons use: merely
opening the side room and looking at an offer stores the viewing merchant's index in it,
without accepting anything, and the [tavern interaction](../operations/0052-tavern-interaction.md)
operation's "Leave" handler is what releases it (writing `0xFFFF`). Switching to another
tavern page releases it; closing the tavern window with a right click does not, and the
offer stays locked - see [Tavern Mission Lock Leak](../bugs/tavern-mission-lock-leak.md).

The index is also not cleared when a mission ends. Observed on a smuggler offer taken and
then failed: the offer came back as a **new letter** pointing at the same task and the
same variable array, its dates refreshed (the descriptor's expiry and variable 1, which
mirrors it, moved about 16 days out; the task's own due date about 10), some of the
mission data re-rolled - and the owner variable still holding the merchant who failed it.
Since the side room accepts its own merchant's index, such a re-issued offer stays
visible to him. The variable is better read as "the merchant this offer is bound to" than
as "who currently holds it".
