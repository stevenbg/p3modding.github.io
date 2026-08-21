# Patrol Letter Crash

## Summary
Opening the personal letters list sometimes crashes the game to desktop while
certain scripted letters are present - most prominently the escort/patrol
mission's "Patrol destination" letters. The crash is long known in the community
as the "patrol mission crash" and looks random: the same letter may crash the
game, show a wrong town in the list, or show no town at all.

## Details
Every [message](../letters.md) carries a town byte that the letters list draws as
its town column, by indexing the 40-slot town-name
[name bank](../ui/name-banks.md) without a bounds check
(`0x0047D928: mov eax, [edx*4+0x6DDA00]`). The resulting pointer goes straight to
the render DLL's text draw, which dereferences it without any guard
(`ddraw_Dll+0xF100`).

The [letter script](../letters/scripted-letters.md) creation command stores the
low byte of a script variable as the town byte, unvalidated (`0x004ED4E4`), and
the patrol script asks it for a variable that does not exist: command 37 of
`patrouille.p2m` - the "Patrol destination" letter - names **variable 131** in a
script that declares 25 variables (see
[Mission Scripts](../letters/mission-scripts.md)). The handler indexes the
variable array with that byte regardless, reading 424 bytes past its end, so the
town byte is whatever heap data follows the array - observed bytes include 40, 95,
228 and 255. It is the only out-of-range letter town variable in any of the game's
94 script files. Drawing such a row reads past the name bank into unrelated
globals, and the outcome depends on the value it hits:

- ids 40..~81 land in the adjacent full town-name table, producing a genuine but
  wrong town name (typically the first town, "Edinburgh");
- a value that points at readable memory usually starts with a zero byte and
  draws as an empty town column;
- anything else - colors, coordinates, small integers - crashes the game the
  moment the list is drawn.

Which globals hold what depends on resolution, loaded mods and session history,
which is why the crash appears intermittent. Only the list is affected: the
letter body and header are formatted at creation through the bounded town-name
helper, so reading a letter is always safe.

## Fix
[mod-fix-patrol-letter-crash](https://github.com/P3Modding/p3-lib/tree/master/mod-fix-patrol-letter-crash)
detours the lookup at `0x0047D928`: town bytes below 40 read the bank as before,
anything else draws an empty string - the same blank town column the unpatched
game shows whenever the wild read happens to survive.
