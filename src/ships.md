# Ships

## Ships Struct
The static ships struct is at `0x006DD7A0`.
The following fields have been identified:
```C
struct __declspec(align(4)) ships
{
  captain *field_0_captains_ptr __tabform(NODUPS);
  ship *field_4_ships;
  convoy *field_8_convoys;
  _DWORD field_C[5];
  class49 field_20_class49_array[16];
  __int16 field_E0_auto_trader_freelist_head; // index of the first free auto-trader record; equals field_F2 when the list is empty
  unsigned __int16 field_E2_unused_ship_index;
  int field_E4;
  __int16 field_E8_unknown_ship_id;
  __int16 field_EA_ship_index_1;
  signed __int16 field_EC;
  signed __int16 field_EE;
  __int16 field_F0;
  unsigned __int16 field_F2_captains_size;
  unsigned __int16 field_F4_ships_size;
  unsigned __int16 field_F6_convoys_size;
  signed __int16 field_F8;
  signed __int16 field_FA;
  unsigned __int16 field_FC_ship_engagement_range_squared;
  signed __int16 field_FE;
  int field_100;
  int field_104;
};
```

## Ship Struct
The pointer to the ships array is stored in the static `ships` struct at offset `0x04`, and the length of that array at offset `0xf4`.

The following fields have been identified:
```
00000000 struct ship // sizeof=0x180
00000000 {
00000000     unsigned __int8 field_0_merchant_index __tabform(NODUPS);
00000001     char field_1;
00000002     unsigned __int16 field_2;
00000004     unsigned __int16 field_4_next_ship_of_merchant;
00000006     __int16 field_6_next_ship_index_in_convoy;
00000008     unsigned __int16 field_8_convoy_index;
0000000A     unsigned __int16 field_A_some_ship_id;
0000000C     unsigned __int16 field_C_next_spotted_candidate;
0000000E     unsigned __int8 field_E_ship_type;
0000000F     unsigned __int8 field_F_maybe_upgrade_level;
00000010     int field_10_capacity;
00000014     int field_14_max_health;
00000018     int field_18_current_health;
0000001C     int field_1C_x;
00000020     int field_20_y;
00000024     ship_route *field_24_route_ptr;
00000028     int field_28_x_delta;
0000002C     int field_2C_y_delta;
00000030     int field_30;
00000034     signed __int16 field_34;
00000036     char field_36_is_on_route;
00000037     unsigned __int8 field_37_unknown_town_index;
00000038     unsigned __int8 field_38_dest_town_index;
00000039     unsigned __int8 field_39_last_town_index;
0000003A     unsigned __int16 field_3A_target_ship_index;
0000003C     char field_3C;
0000003D     char field_3D;
0000003E     signed __int16 field_3E_maintenance;
00000040     unsigned __int16 field_40_crew;
00000042     unsigned __int16 field_42_captain_index;
00000044     int field_44_timestamp2;
00000048     int field_48_maybe_calculated_arrival_timestamp;
0000004C     int field_4C_timestamp;
00000050     unsigned __int16 field_50_spotted_ships[2];
00000054     unsigned int field_54_wares[24];
000000B4     float field_B4_avg_prices[24];
00000114     int field_114_payload_buy_sum;
00000118     int field_118_maybe_used_capacity;
0000011C     int field_11C_equipment_weight;
00000120     int field_120_arty_stuff;
00000124     int field_124;
00000128     int field_128;
0000012C     int field_12C;
00000130     __int16 field_130;
00000132     unsigned __int16 field_132_route_stop_index; // current stop in the trade route stop pool, see Trade Routes (.rou)
00000134     __int16 field_134_status;
00000136     char field_136;
00000137     char field_137;
00000138     unsigned __int16 field_138_docking_counter;
0000013A     char field_13A;
0000013B     char field_13B;
0000013C     char field_13C_artillery[24];
00000154     int field_154_cutlasses;
00000158     int field_158;
0000015C     char field_15C_is_pirate;
0000015D     unsigned __int8 field_15D;
0000015E     __int16 field_15E;
00000160     char field_160_ship_name[32];
00000180 };
```

## Crew, Cutlasses and the Equipment Weight
Three fields describe what a ship carries besides cargo, all verified in-game by
changing one thing at a time and diffing the struct:

|Field|Meaning|
|-|-|
|`field_40_crew`|sailors aboard; `0x005184F0` derives the complement as `max(20, capacity/2000 + upgrade_level * class_factor)`, the class factors being 3, 5, 8, 10 (first dword of the per-class blocks at `0x0066E030`, stride `0x18`). `capacity/2000` is the capacity in loads, so a ship's crew is roughly its load capacity.|
|`field_154_cutlasses`|cutlasses aboard (they are not [ship weapons](./reference/ship-artillery.md))|
|`field_11C_equipment_weight`|the cargo space all of that occupies|

`field_11C_equipment_weight` is the sum of three terms:

```
400 * max(0, crew - class_base) + 10 * cutlasses + sum(weapon scaling factors)
```

The class base is the per-class reference crew at `0x00673664` = 10, 16, 30, 24 - crew up
to that allowance is free, and only sailors above it cost hold space. Measured on a
type 1 ship (base 16) with 34 crew, 49 cutlasses and 4 large + 2 small catapults:
`400*18 + 10*49 + (4*2000 + 2*1000)` = 17690, exactly the value in the field. A captured
type 2 hull (base 30) carrying only 8 crew, 27 cutlasses and 7 bombards + 2 small
ballistas gave `0 + 270 + 16000` = 16270, also exact - so an under-crewed ship gets no
credit for the unused allowance. Free cargo
space is therefore `field_10_capacity - field_11C_equipment_weight - loaded wares`,
which is the arithmetic the pirate AI uses when it checks whether it has room for loot.

`0x005182B0` is where the game maintains all of this. Called on a ship it floors every
ware's cargo to a whole unit, then recomputes `field_114` (the payload's purchase value),
`field_118` (used space), `field_11C` by exactly the formula above - `0x0051838A` for the
crew term, `0x00518411` for the cutlasses term - and `field_120`, and **returns the free
cargo space**. That return is the budget an
[auto trade route stop](./ships/trade-routes.md) spends on loading.

## Speed
`0x00612930` computes a ship's current speed, and it is the number the
[pirate AI](./pirates.md) compares when it decides whether it can run a target down. It
returns 1 for a ship with no capacity or no maximum health, otherwise:

```
base[ship_type & 3]                                    // 693, 693, 578, 578 at 0x0067ADC0
  * (4096 - 614 * used_capacity / capacity) / 4096     // a full hold costs 15%
  * clamp(179 * health / max_health + 113, 166, 256) / 1024
  * (2550 + navigation_skill) / 2550                   // the captain
```

- A **full hold** costs about 15% of the ship's speed.
- **Damage** costs up to about 35%, and the clamp means anything above roughly 80% of
  maximum health gives the full term - which is also the threshold at which a damaged
  pirate breaks off and sails home.
- The **captain's navigation skill** is worth up to +10% (skill 255; the displayed level 5
  is skill 215, so +8.4%, each level of 43 points being +1.7%). A ship with **no** captain
  skips the factor entirely and so matches a navigation-0 captain: captains never make a
  ship slower.

## Iterating One Merchant's Ships

`field_4_next_ship_of_merchant` chains every ship of one owner, and the head of that
chain is `+0xE` of the merchant record. The merchant array is at `game_world + 0x78`
with stride `0x650`; the accessor `0x005303C0` (thiscall on the game world, one
argument) computes `[this+0x78] + index * 0x650`.

The game's per-merchant ship census at `0x004F0AB1` walks it: fetch the merchant
record, take `+0xE`, then follow `+0x4` while the index stays below the ship count at
`[0x006DD894]`. Iterating one player's fleet this way costs his ship count rather than
the world's - worth having when a late game holds a thousand ships.

## Ship Status

`field_134_status` takes values from `0` to at least `0x15`. Two of its classes are
established, and the game itself tests for exactly them side by side in the
per-merchant ship census at `0x004F0B02`/`0x004F0B11`/`0x004F0B26`:

|Test|Meaning|
|-|-|
|`status <= 3`|the ship is at the town in `field_39_last_town`, not at sea|
|`status == 0xF`|merchant vessel at sea|

Within the in-port family, `0` is a ship lying in the port and `3` is set while it
enters one - at `0x004E13FA`, which also clears the convoy fields `+0x6`/`+0x8` and ORs
`0x60` into the flags at `+0x3C`. `field_39_last_town` already names the town at that
point, and this is the state in which the town becomes enterable: a save with one ship
sailing to Rostock showed the ship flipping from `0xF` to `3` exactly when Rostock's
town view and tavern became reachable, still before docking.

Nothing town-side marks that transition. A byte-exact snapshot of all 24 town structs
taken while the ship was at sea, diffed the moment Rostock became enterable, shows no
change at all in Rostock (the other towns differ only in economy fields) - so a town
carries no "enterable" flag and no list of the ships present. Ship status is the
whole answer.

`0x12` is an AI pirate vessel at sea; `mod-scrollmap-render-all-ships` draws exactly
`0xF` and `0x12`.
