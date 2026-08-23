# Reference Data
The enums, tables and structs that the rest of the book keeps referring to. Nothing here
describes behaviour - it is the vocabulary the behaviour chapters are written in.

|Page|Holds|
|-|-|
|[Ware Types](./reference/wares.md)|the 24 ware ids and the barrel/bundle scaling that converts raw units to displayed ones|
|[Buildings](./reference/buildings.md)|building ids, the "new building" ids operations use, and the table mapping one to the other|
|[Facilities](./reference/facilities.md)|the facility ids a town's businesses are grouped into, and the `facility` struct|
|[Storage](./reference/storage.md)|the `storage` struct that both towns and [offices](./merchants/trading-office.md) begin with|
|[Ship Types](./reference/ship-types.md)|the four hull classes|
|[Ship Artillery](./reference/ship-artillery.md)|weapon types, their scaling factors, combat power and the slot layout|

## Functions
| Address | Signature | Description |
| - | - | - |
| `0x0064F7B9` | `malloc_wrapper(size_t size)` | A thin wrapper around malloc that is used for most heap allocations of the game. |
