# Footprint bars of whole hours from hours added together (2026-10-14)

Karl: zoomed out from Oct 8 to now, the bar statistics (they read the footprint) "fill in slowly from the right side to the left".

## What it was

The page asks for the footprint in slices of at most `FOOTPRINT_BUDGET` (5,000) instrument-minutes, newest first, a minute read back from
the store counting four. With thirteen markets on 1 h candles, a slice covered about six hours of the last day and two hours before it:

- On the live server (read-only GETs, 13 markets, Oct 8 to now): **66 answers, 22.1 s**, each read back answer 0.5 to 1.1 s. A second walk
  straight after took 28 s: nothing was kept between zooms (the kept read-back hours hold 20,000 minutes; the window is about 112,000).
- Reading a minute back, off the server (the live database opened read-only, 14 BTC markets, one day): SQL 30 µs, `JSON.parse` 36 µs,
  making its map 12 µs: 78 µs a minute. So most of the 22 s on the server was waiting behind its other work between 66 round trips.

## What changed

`FootprintRecorder` keeps each instrument's ended hours added together (`#hourRows`: [bin, buy, sell] in a `Float64Array`, on the step
they were recorded at). Bars of whole hours (1 h, 4 h, 1 d) read one entry an hour; minutes, statistics and size-line sums still come
from the minutes' sums held in memory. An hour is dropped when a minute of it changes (`#rowsToChange`, the one way in for a trade or an
order) and at the retention cutoff. An hour only the store holds is added up from the stored rows as they come (no per-minute map, not
kept read back).

The route first calls `prepare`, which adds up the hours the answer will read, newest bar first, letting the event loop go every 8 ms, up
to four budgets of minutes read and always the whole newest bar (a daily bar of fourteen markets read back no longer merges 80,640 weighted
minutes in one go: round 10 of the review). In `#reach` a held hour costs 2.

## Numbers (off the server, the live recording read-only, 13 markets, 1 h bars from Oct 8, budget 5,000)

| | answers | total | worst answer | event loop held at most |
| --- | --- | --- | --- | --- |
| before, first look | 129 | 8.13 s | 285 ms | 285 ms |
| before, repeat | 129 | 7.53 s | 357 ms | 357 ms |
| after, first look | 17 | 9.28 s | 1,225 ms (prepared in turns) | 201 ms |
| after, repeat | 1 | 0.18 s | 185 ms | 196 ms |

The first look reads every minute once, as before (about 6.6 s of it is the SQL and the parsing); it comes in 17 answers instead of 129.
After that, a zoom over the same days is one answer. Bars of less than an hour (5 m, 15 m) are read minute by minute as before.
