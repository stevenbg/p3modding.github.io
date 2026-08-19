# Name Banks
Three consecutive 40-slot string-pointer banks in BSS hold localized name lists,
loaded from one text blob of consecutive NUL-terminated strings:

|Bank|Address|Content|
|-|-|-|
|A|`0x006DD8C0`|tavern names|
|B|`0x006DD960`|guild names|
|C|`0x006DDA00`|town names|

The populate function at `0x00512730` (thiscall; this = loader object: `+0` blob
start, `+4` blob end, `+8` per-slot pointer array) fills bank C's 40 slots first,
then bank B's, then bank A's, walking the blob string by string. When the list
has fewer entries than slots (a 24-town map, for example), the remaining slots
keep pointing at the blob's first string. Further writers at `0x00511E76`,
`0x00511EB1`, `0x005127FC` and `0x0051288C` repopulate banks at runtime from a
cache object.

Directly after bank C, at `0x006DDAA8`, sit 40 consecutive town-name pointers
covering all towns of the full map - fields of a larger name-registry object at
`0x006DDAA0`. That object's getter `0x00512B20` (thiscall(this, id)) is NOT a
town lookup: it resolves ids through two offset-table sections (`this+0xC8`/
`+0xCC` into blobs at `this+0xB4`/`+0xB8`, section limits `this+0xD8`/`+0xDA`)
and returns dynamic names - id 15 resolved to a ship name in testing. Town names
by savegame town index come from bank C.

Bank consumers index the 40 slots directly, e.g.
`mov eax, [index*4 + 0x006DDA00]` - usually with an index that is a genuine town
byte. The letters list does it with an unvalidated byte, which is the
[patrol letter crash](../bugs/patrol-letter-crash.md).
