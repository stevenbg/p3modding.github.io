# Town View Fill Out of Bounds

## Summary
The flood fill that rebuilds a town map's region matrix when the town view opens does not
bounds-check its coordinates. A seed placed off the map edge makes it read - and write
`0xFFFF` - below the matrix allocation: a crash-to-desktop when the heap places the matrix
against a no-access page, and silent corruption of whatever lies before the block otherwise.
The silent case is the common one.

Captured once in the wild (crash report analysed below): clicking Ripen on the world map
while a convoy was leaving its port.

## The rebuild chain
The [local map scene](../ui.md#the-local-map-scene) keeps a region object per map in the
array at `[0x006DF270]` - records of `0x1024` bytes indexed by the map id, the object
pointer at `+0x8`. The scene populate pass (`0x00595CA0`, called from the loader at
`0x0058A8B5`) rebuilds the object when its byte `+0x7A` is 0, via `0x0062C730`:

1. **The matrix.** A `u16` per tile at `object+0x59C`, `malloc(stride * height * 2)`,
   allocated lazily at `0x0062B8AC`; `stride` is the scene's `+0xC314`, `height` its
   `+0xC318`. A byte map of the same grid sits at `object+0x70` (allocated `stride *
   height` at `0x0062C887`).
2. **Seed selection** (`0x0062C340`). The scene carries a list of map coordinates at
   `+0xC6AC` (8-byte x/y dword pairs, count at `+0xC698`). The routine picks the entry
   nearest `(object+0x36, object+0x38)` by the isometric metric `(2*dx)^2 + dy^2`, stores
   it at `object+0x5AA/+0x5B6`, then gathers every entry within distance-squared `0x4C9`
   of it as further seeds (count `object+0x5D4`).
3. **Seeding** (`0x0062DD70`, called at `0x0062CD4C`). Each seed is stepped one tile in
   four diagonal directions with `0x0058B740` and the fill is started from each stepped
   position.
4. **The fill.** `0x0062BD50(this, x, y)` marks the cell `0xFFFF`, then probes a ring
   of cells around it: the coordinate is pre-stepped with `0x0058B740` (direction bytes
   at `0x0067792F`, then `0x00677929`), and eight deltas from the i16 tables at
   `0x0067B610`/`0x0067B620` - `(1,0), (0,2), (-1,0), (0,-2)`, then `(1,2), (-1,2),
   (-1,-2), (1,-2)` - are added **cumulatively** to the walking position
   (`0x0062BE26`/`0x0062BE2E`), four per ring (`add bl,4 / cmp bl,8` at `0x0062BE63`).
   Probed cells matching the region value recurse through `0x0062C0A0(this, x, y)`,
   which walks the same tables (`0x0062C2BA`) and calls itself at `0x0062C2AD`.

`0x0058B740(class22, &x, &y, direction)` steps one tile on the staggered isometric grid.
The direction codes are the byte table at `0x00677928`: `0x00, 0x20, 0x40, ..., 0xE0` for
N, NE, E, SE, S, SW, W, NW - N is `y -= 2`, NE is `x += y & 1; y -= 1`, NW is
`y -= 1; x -= y_new & 1`, and so on.

**And it already bounds-checks the result.** Its tail tests both axes against the scene's
grid and returns the answer:

```asm
0058b7be  mov  edx, [x]
0058b7c0  mov  esi, [ecx+0xC314]     ; stride
0058b7c6  cmp  edx, esi
0058b7ca  jae  fail                  ; x >= stride, or x < 0 (unsigned)
0058b7cc  mov  eax, [y]
0058b7ce  mov  edx, [ecx+0xC318]     ; height
0058b7d4  cmp  eax, edx
0058b7d6  jae  fail                  ; y >= height, or y < 0
0058b7d8  mov  eax, 1                ; in bounds
0058b7dd  ret  0xc
0058b7e0  fail: xor eax, eax
```

## The bug
Nothing in steps 3-4 checks bounds. The stepper never clamps, and both fills index the
matrix with `stride * y + x` directly:

```
0062BD77  mov ax, [edx+ecx*2]      ; edx = matrix, ecx = stride*y + x
0062BD8A  mov word [ecx], 0xFFFF   ; ecx = matrix + index*2
```

A seed within one row of the map edge steps to `y = -1`, giving a negative index. The
read at `0x0062BD77` crashes when the matrix allocation happens to start right after a
no-access page; when the memory below is readable, the write at `0x0062BD8A` corrupts two
heap bytes below the allocation instead, and the fill keeps walking. The neighbouring
routine `0x0062B860` - the one that allocates the matrix - does check its row argument
against `+0xC318` (`0x0062B896`) before touching the same matrix; the two fills have no
such check.

**The seeder discards that answer.** Every one of its four step-then-fill pairs overwrites
`eax` two instructions after the call and fills unconditionally:

```asm
0062ddf1  call 0x0058B740      ; returns 1 = in bounds, 0 = off the map
0062ddf6  mov  eax, [esp+0xc]  ; the answer is gone
0062de02  call 0x0062BD50      ; fill anyway
```

So the engine computes exactly the predicate that would prevent this bug and throws it away.

An off-map seed does not write once and stop: the fill marks the cell, then probes its
ring of neighbours and recurses into matching ones, so one bad seed can wander further
out of bounds. Nothing in that walk - the pre-steps, the cumulative deltas, the
recursion - is clamped either, so a region touching any border walks off it even from a
valid seed.

Two things are separately conditional, which is what makes this bug look random:

- **which** list entry seeds the fill, since the selection is nearest-to-anchor plus
  everything within ~35 tiles - so an edge waypoint is only used when the anchor is near it;
- **whether** a bad seed crashes or corrupts, which is down to heap layout.

The coordinate itself is not random, and the shipped map files say why. Section 0 of
[`<id>.nodes`](../file-formats/nodes.md) is the ships' docking path; its points convert to
tiles as `tile_y = (py - 10) / 11` and, on even rows, `tile_x = (px - 17) / 34`. Ripen
(`7.nodes`) carries 23 of them on tile **row 0**, one at `px = 3485` = tile **(102, 0)**, and
the `0xE0` step from there is `(101, -1)` - exactly the crash registers below, and exactly
what the guard logged twice in one session.

**Nineteen of the twenty-four towns have such row-0 waypoints**, from Hamburg's 17 to
Reval's 36. That is expected: ships enter from the sea at the map's northern border, so the
docking path hugs row 0. Only London, Rostock, Aalborg, Stockholm and Torun have none - their
harbours open on a side edge instead, where the out-of-range step is in `x` and wraps onto the
neighbouring row rather than leaving the block. So this is a defect of the engine and of
essentially every map, not of one town.

Those two events say nothing about how often the bug crashes - the guard stopped them before
they touched the matrix. What does argue that **the usual outcome is silent corruption** is
the rate: the guard caught two events in a single session, where the crash has been seen once
in months of play. The seeding is unchanged by the guard, so the event rate is the same as it
always was, and only a small fraction of those events can have been crashing.

What clears `+0x7A`, what the anchor tracks, and what the marked region is used for have
not been pinned down.

## The captured crash
From `_crash_report.txt` (mod-crash-reporter): access violation at `0x0062BD77`, reading
`0x1075AFD2`. Registers: matrix `edx = 0x1075B008`, index `ecx = 0xFFFFFFE5` = -27,
`esi = x = 101`; with the map's stride 128 that is exactly `128 * (-1) + 101` - the fill
was called at `y = -1`. The matrix allocation began 8 bytes past a page boundary and the
page below (`0x1075A000..0x1075B000`) was committed `PAGE_NOACCESS`, so the read at
`matrix - 0x36` faulted instead of corrupting.

## Fix
`mod-fix-harbor-fill-oob` detours both fill entries (`0x0062BD50`, `0x0062C0A0`) to a
guard that returns immediately - `ret 0x8`, like the originals, whose return value no
caller reads - for any coordinate outside `[0, stride) x [0, height)`. An in-range fill
is unchanged, and guarding `0x0062C0A0` covers the recursion. Rejections are logged, so
every log line is a crash or corruption that did not happen.

The guard re-implements the test `0x0058B740` already performs, which invites a smaller fix -
honour the stepper's return value with a `test eax,eax / jz` after each of its four calls in
`0x0062DD70`. That does not fit: the four sites are packed tight, with the fill's own argument
loads (`mov edx,[esp+0xc]` / `mov eax,[esp+0x10]`) immediately after each call and no padding
anywhere in the routine, so the four bytes have nowhere to go without a detour to make room -
at which point nothing is saved over guarding the fill itself. It would also leave the
recursive fill unprotected, since that never consults the stepper.
