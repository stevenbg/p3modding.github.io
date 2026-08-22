# Ship Artillery
Ship Weapons are represented by the following enum:
```rust
pub enum ShipWeaponId {
    SmallCatapult = 0x00,
    SmallBallista = 0x01,
    LargeCatapult = 0x02,
    LargeBallista = 0x03,
    Bombard = 0x04,
    Cannon = 0x05,
}
```
Cutlasses are not ship weapons.

## Ship Artillery Scaling
The amounts P3 displays in-game are not the values the game uses under the hood.
Every ship weapon has a scaling factor, through which the game divides the actual values.

A table that maps every `ShipWeaponId` to its scaling factor can be found at `0x00672CB4`.
This reveals the following factors:
```rust
ShipWeaponId::SmallCatapult => 1000
ShipWeaponId::SmallBallista => 1000
ShipWeaponId::LargeCatapult => 2000
ShipWeaponId::LargeBallista => 2000
ShipWeaponId::Bombard => 2000
ShipWeaponId::Cannon => 1000
```

## Combat Power
Next to the scaling table sits a second, byte-wide table at `0x00672CC8` giving each
weapon a combat power:

```rust
ShipWeaponId::SmallCatapult => 9
ShipWeaponId::SmallBallista => 10
ShipWeaponId::LargeCatapult => 22
ShipWeaponId::LargeBallista => 24
ShipWeaponId::Bombard => 30
ShipWeaponId::Cannon => 18
```

Fitting a weapon (`0x0051A4E0`) adds its scaling factor from `0x00672CB4` to the ship's
`field_11C` - the capacity the guns occupy - and its power from `0x00672CC8` to the
ship's `field_120`, so `field_120` is the ship's total artillery power. Removing a
weapon subtracts both. That total is one of the two halves of the fighting strength the
pirate AI compares before attacking; the other is the crew count in `field_40`.

Two further six-byte tables sit in the same block and are not yet identified:
`0x00672CC0` = 32, 32, 77, 77, 96, 58 and `0x00672CD0` = 60, 80, 60, 80, 90, 90.

## Ship Artillery Slots
A ship's artillery slots are filled with the following enum:
```c
enum ship_artillery_slot : unsigned __int8
{
  ship_artillery_slot_small_catapult = 0u,
  ship_artillery_slot_small_ballista = 1u,
  ship_artillery_slot_large_catapult = 2u,
  ship_artillery_slot_large_ballista = 3u,
  ship_artillery_slot_bombard = 4u,
  ship_artillery_slot_cannon = 5u,
  ship_artillery_slot_large_neighbor = 6u,
  ship_artillery_slot_unavailable = 7u,
  ship_artillery_slot_empty = 255u,
};
```

The slots are indexed as indicated here:
<p style="text-align:center">
    <img src="ship-artillery-slots.png">
</p>
