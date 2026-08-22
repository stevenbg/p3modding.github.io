# Set Trade Route Active
Operation `0x68` activates or deactivates a ship's trade route - the [ship panel](../ui/ship-panel.md)'s
"active" checkbox.

|Offset|Type|Value|
|-|-|-|
|0x00|u32|opcode `0x68`|
|0x04|u32|ship index|
|0x08|u32|active: 0 deactivates, anything else activates|

The handler at `0x0053DF00` (operation switch case `0x5364C5`) validates the ship
index, resolves the convoy, and updates the route state flags on the ship (`+0x136`,
`+0x3D`) or convoy. Deactivating also resets the current destination to the last
visited town.

`transfer_loaded_traderoute` (`0x005492D0`) enqueues the deactivation as its first
step when replacing a ship's route (`0x005494DD`).

Deactivating does not remove any stops; see
[Trade Route Stop Town Change](./006a-trade-route-stop-town-change.md) for that.
