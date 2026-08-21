# Tavern

The side room's missions are not a structure of the tavern's own: each offer is a
type-`0x71` letter in the visiting merchant's letter chain, paired with a scheduled task
holding the mission's script variables. See
[Tavern Missions](../letters/71-tavern-missions.md).

The tavern window itself is documented under [UI](../ui.md): vtable `0x00679B78`, its
object in the static `0x006E5574`, the selected page at `window + 0x1BF4` and the town at
`window + 0x1BFC`. The side room is page `9`, drawn by `0x005D7FD0`.
