# The aggregated footprint, and how the footprint loads (2026-10-13)

A feature request: candles from one market (Binance perpetual) with the footprint, the bar statistics, open interest and NL/NS added up over
all venues, as MMT does. The candles, the heatmap, the flow column and the Delta pane already were; the footprint and the bar statistics were
the chart market's alone. This is the footprint part (open interest and NL/NS need data the app does not read yet).

## The question

`FootprintRecorder.queryMany(ids, …)` adds markets together on one grid (rows of the requested size rounded to a multiple of the coarsest
recorded step; every market's rows placed by their low price): a bar's minutes are those any market recorded, its size statistics and size
lines are given only when every market-minute had them. `query(id)` is `queryMany([id])`. The page asks for the markets of the ALL VENUES
lines (`aggregateIds`), so the bar statistics' delta and the Delta pane's count the same markets.

## What it cost, and what changed

Measured on a copy of the live recording (14 markets with footprints, a week; `.tmp/fp-agg-bench.mts`), one question for 200 bars:

| Bars | 14 markets, first look | again | one market |
| --- | ---: | ---: | ---: |
| 1 m | 202 ms | 64 ms | 3 to 11 ms |
| 5 m | 216 ms | 193 ms | 9 to 12 ms |
| 15 m | 2.1 s | 0.6 s | 26 ms |
| 1 h | 8.3 s | 8.6 s | 0.4 s |

A request merges on the server's only thread, and the page asked for its whole window again every 5 s: one market's week of hourly bars was
already 0.4 s of every 5 s, fourteen would have been more than the 5 s. Two changes, for the single market as much as for the aggregate:

- **Settled bars are kept.** A bar that ended two minutes ago (by the source's clock) does not change; every 5 s only the bars from there on
  are asked for again, and a view wholly in the past is not asked for again at all.
- **Slices.** A question may carry a budget of instrument-minutes (`budget`, 5 000 from the page); the answer takes whole bars newest first
  up to it, says where it starts (`from`), and the page asks for the next older slice until the view is covered. A minute read back from the
  store counts four (the SQL and the parsing of its rows cost about three times the merge; a CPU profile of the cold case: store reads 13 %,
  row parsing 6 %, the merge 10 %, garbage collection 16 %). A source without budgets answers the whole window, as before.

The same 200 bars, now (worst slice is what blocks the server at once):

| Bars | 14 markets: slices, worst, total | one market: slices, worst, total |
| --- | --- | --- |
| 1 m | 1, 169 ms, 169 ms | 1, 9 ms |
| 5 m | 3, 58 ms, 152 ms | 1, 6 ms |
| 15 m | 22, 168 ms, 1.5 s | 2, 16 ms, 21 ms |
| 1 h | 125, 188 ms, 4.6 s | 8, 60 ms, 0.24 s |

After the first look the refresh is the open bar: a few milliseconds. A change of row size (zooming the price axis) or of the size lines is
another question and loads the view again in slices.

## Not done

- Rows of markets whose prices differ (USD spot against USDT perpetuals, 10 bp or more) are not shifted onto the chart market's prices: the
  panel says to choose Perp in the filter.
- Loading the recorder at start reads a week of footprint minutes synchronously (6.5 to 9.7 s on this copy): the likely cause of the server's
  stall in its first seconds, not looked into yet.
