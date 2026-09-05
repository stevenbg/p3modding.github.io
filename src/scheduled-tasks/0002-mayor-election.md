# Mayor Election

Scheduled task opcode `0x02`, handler `0x004DBB00`. There is one task per town: its data
holds the town's **election day of the year** at `+0x8` (u16) and the town index at `+0xE`.
A day of 365 or more counts as unset and is replaced by `(due_timestamp >> 8) % 365` when
the task first runs.

The handler compares that day with today's day-of-year (`[0x006DE4A4]`; the year is
`[0x006DE4A2]`):

- equal: the election runs, `0x00528E60(town)`, and the task is re-armed for the same day
  next year;
- already past: re-armed for next year, no election;
- still ahead: re-armed for this year.

The new due timestamp is `((year * 365 + day) << 8) | 0xFA`, so the election is resolved in
the last ticks of its day.

## Candidates: `0x00528CF0(town, out[4])`

The ballot has four slots, highest score first. A merchant enters if all of these hold:

- the **candidature flag** `merchant + 0x118` is set (`0x004F9360`). A human's flag defaults
  to `1` (constructor `0x004F72D3`) and is toggled by operation `0x45` (`0x004F9390`), the
  town hall's "Accept candidature?" question. An AI merchant always stands, unless its
  control word has bit `0x4`, in which case the flag is cleared instead;
- the merchant is a **guild member in that town** (bit `town` of `merchant + 0x468`, set by
  [Join Guild](../operations/0037-join-guild.md));
- the town is his **hometown** (`merchant + 0x19`);
- his **rank there is Patrician or above** (`merchant + 0x39C + town >= 5`, see
  [Ranks](../merchants/ranks.md)).

The score is the merchant's [reputation](../merchants/reputation.md) **in that town**,
`0x004F9580`: the float at `merchant + 0x2FC + town*4` truncated to an integer, or `0x7FFF`
once it exceeds `32758.0` (`[0x00672F58]`).

The slots start out holding the town's four **notables** (ids `0xFF`..`0xFC`; notable `n`
has id `0xFF - n`), with fixed scores from the byte table `0x006737B4` = `59, 65, 70, 80`:
notable `n` scores entry `n`, and the initial ballot is notable 3 (80), 2 (70), 1 (65),
0 (59). A merchant is inserted in sorted position and the lowest occupant falls off, so he
is listed only if his reputation exceeds the lowest remaining notable, and with `k`
merchants standing the notables left are the top `4 - k` of 80/70/65/59.

Each slot also carries four bytes the town hall page renders: `out + 4 + slot`
(`0x004F9430(merchant)` or `town + 0x6E0 + n`), the rank at `out + 8 + slot`
(`merchant + 0x39C + hometown` or `town + 0x6E4 + n`), and `out + 0xC + slot` /
`out + 0x10 + slot` (`merchant + 0x1C` / `+0x1D`, or `town + 0x6D4 + n` / `+0x6D8 + n`).

## The vote: `0x00528E60`

1. The four scores are read again (merchants through `0x004F9580`, notables from the table).
2. If **no merchant** stands, each notable gets a date-seeded bonus:
   `((days * (slot + 7)) % 67) & 0x1F` with `days = [0x006DE4B4] >> 8`, or a flat 31 for
   the score of exactly 80.
3. **Bribed councillors** (`town + 0x6DC..0x6DF`, the briber's merchant id per councillor,
   written by [Bath House Bribe Success](../operations/0042-bath-house-bribe-success.md)):
   each slot naming a merchant on the ballot gives him **3 votes**, is reset to `0xFF`, and
   takes 3 votes out of the pool, which starts at **120**. Bribes by merchants not on the
   ballot are left standing.
4. The rest of the pool is split by score: `votes[s] += score[s] * pool / sum(scores)`,
   integer division.
5. The highest total wins; a tie goes to the earlier slot.

With the list sorted by reputation, the top name wins unless bribes overturn it. Four
candidates share the pool at roughly 25..35 votes each, so the 12 votes four bribed
councillors are worth can decide a close race.

## The result

- **Re-election** (winner is the sitting mayor `town + 0x6F1`): the consecutive-term counter
  `town + 0x6F3` increments, capped at 250. When it reaches 1 for a merchant mayor, a task
  `0x38` (handler `0x004ECB14`, town at `+0x8`, mayor at `+0x9`) is scheduled
  `186 - 93 * winner_votes / (total_votes + 1)` days ahead; it runs only if that merchant is
  still the town's mayor.
- **New mayor**: `town + 0x6F3 = 0`, the winner's id is written to `town + 0x6F1`
  (`0x005290BE`), and the previous mayor, if a merchant, has his ranks and reputation
  recomputed (`0x004F75F0`).
- In **every town** the notables' rank bytes `town + 0x6E4..0x6E7` are rewritten
  `5, 5, 4, 4` (two Patricians, two Councillors); a notable holding a mayor's seat gets `6`,
  or `7` if he is the alderman (`0x00529149`, `0x0052913F`) - the rank scale continues past
  Patrician with 6 for a mayor and 7 for the alderman.
- A merchant winner has his ranks and reputation recomputed as well. A **human** winner
  receives note `0x18` through the [note creator](../letters/ship-notes.md)
  (`create_note(mayor, 1, 0, town, 0x18, town)`), and if he is the local player
  (`[0x006DFC14]`) the [event window](../ui/event-window.md) announcement of type `5`.

Since candidacy needs Patrician rank **in the town itself** and the score is the town-local
reputation, moving the home office to a new town does not bring its seat within reach on
the merchant-wide reputation terms alone: the local terms have to be built up there first,
and the seat is only contested on the town's election day.

## Appointment without an election: operation `0x46`

`op + 4` merchant, `op + 8` town; handler `0x00535FD6`, the dispatcher's switch case (the
compiler also left an unreferenced copy at `0x0053AEC0`). Accepted only if the seat is **not held by a merchant** (`town + 0x6F1 >=
merchant count`: a notable, or `0xFF`) and the merchant's rank in his hometown is at least
Patrician - or `op + 0x10` holds `0xBADEAFFE`, a token that waives the rank check but not
the vacancy check. The vacancy task below passes it (`0x004EA1D9`), as do the debug menu's
two "make mayor" commands (WM_COMMAND `0x8035`/`0x8052`, handlers `0x0041AD50`/`0x0041B520`).
The handler then writes `town + 0x6F1`.

Its regular producer is town task `0x2F` (`0x004EA0D4`, in towns with a Market Hall; it
re-arms every `0xE00` ticks = 14 days while a notable holds the seat, daily otherwise): a
seat held by a notable is handed to the **AI** merchant of highest hometown rank who is not
mayor elsewhere - any rank above 0 will do, since its operation carries the token. Human
merchants are never picked, and the task does nothing in a network game (`[0x006DF31C]`).
The other writers of
`town + 0x6F1` are the found-settlement mission (`0x004EC00D`, the founder becomes mayor),
world initialization and loading (`0x00528705`, `0x005288B2`) and a vacate path writing
`0xFF` (`0x0053A958`).

## The town hall's Town info page

The page's "nominated to date" list comes from the same `0x00528CF0`, called at
`0x005E3358` in the town hall window (town index at `window + 0x18E8`), so it shows the
ballot in vote order - the merchants and notables who would compete if the election were
held now.
