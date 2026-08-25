# Storage
The `storage` struct contains a town's or an office's current wares and related data.
Some of the fields have been identified:
```
00000000 ; derived from post-malloc initialization of office and town
00000000 storage         struc ; (sizeof=0x2C0, mappedto_146)
00000000                                         ; XREF: town/r
00000000                                         ; office/r
00000000 field_0         dw ?
00000002 field_2         dw ?
00000004 field_4_current_wares dd 24 dup(?)
00000064 field_64_daily_consumptions_businesses dd 24 dup(?)
000000C4 field_C4_daily_production dd 24 dup(?) ; ACTUAL output per day, staffing-scaled; a daily accumulator the owner's tick zeroes. On a town this is the market hall's "Town" column, on an office its "Trader" contribution - see Production
00000124 field_124_ship_weapons dd 6 dup(?)
0000013C field_13C_prod_time_series storage_production_time_series 24 dup(?) ; one 0x10-byte record per ware, all 24: eight u16 slots written at index ([0x006DE4B4] >> 8) & 7. Wares 0x00..0x13 are written by the ware producers, and the four militia weapons at 0x27C..0x2BB by the Weaponsmith alone (0x0050FB1D) - see Production
000002BC field_2BC_cutlasses dd ?
000002C0 storage         ends
```