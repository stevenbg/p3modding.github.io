# Game Strings

The game's string class is a single pointer to character data with an MFC-style
reference-counted header in the 12 bytes before it:

```
block+0x0   refcount  (interlocked)
block+0x4   length
block+0x8   capacity
block+0xC   the characters (what the string object points at), NUL-terminated latin1
```

A default-constructed string points at the shared empty block: `0x006C7CD0` holds the
nil block's address, so an "empty string object" is the dword `[0x006C7CD0] + 0xC`.

## The methods

| routine | what it does |
|-|-|
| `0x0064EFC8` | copy constructor: **shares** - takes the source's data pointer and `InterlockedIncrement`s the refcount at `data-0xC` (falls back to a copy for a non-shareable source) |
| `0x0064F390` | assign from a C string: allocates/reuses the destination's own block and **copies** the characters (strlen `0x0063B6FA`, bounded copy `0x0064F734`) |
| `0x0064F253` | release: skips the nil block (compared by address against `[0x006C7CD0]`), `InterlockedDecrement`s `data-0xC`, and frees the block when the count reaches zero |
| `0x0064F142` | the free, dispatched by the block's capacity class: `0x40` and `0x80` blocks go back to dedicated pools (the first at `0x006EE1A8`), other sizes to the general allocator |

## The by-value calling convention - the trap

Several routines take a game string **by value** (one pointer slot on the stack) and
**release it before returning**. Verified for both ticker posters:

- `0x0042B6A0`, the event-ticker enqueue (the top-left boxes): copies the text into the
  slot's own string, then releases the argument at `0x0042B8F9`;
- `0x0042BB20`, the letter/notice poster (the top-right boxes): same convention.

Passing a bare byte buffer where such a routine expects a string therefore executes
`InterlockedDecrement(buffer - 0xC)` - a write into whatever heap block precedes the
buffer - and a wild pool-free whenever that garbage dword happens to reach zero. This
exact mistake in a mod produced a long-lived, hard-to-trace heap corruption (d3d9's
resource list and a Windows CoreMessaging object were among the victims) before it was
caught by a hardware write watch.

The safe pattern for calling such a routine: start from the nil string
(`[0x006C7CD0] + 0xC`), build the content with the assign `0x0064F390`, and hand the
result over - the callee's release then balances the ownership exactly, freeing through
the game's own pools.
