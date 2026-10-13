# Open interest added together, and NL/NS over venues (2026-10-13)

The second and third parts of the request for an aggregated workflow (the first: `footprint-aggregated-2026-10-13.md`). The OI pane, the bar
statistics' oi chg row and NL/NS were one market's: the chart market's, or Binance's standing in. The app read open interest from Binance
(history and live) and Hyperliquid (live) only.

## What each venue publishes (BTC, measured 2026-10-13)

| Venue | Open interest | History | Page can read it |
| --- | ---: | --- | --- |
| Binance | 94K BTC | 5 min bars, 30 days | yes |
| Bybit | 57K BTC | 5 min bars, pages back past a week | yes |
| Hyperliquid | 38K BTC | none (the server records its pushed asset contexts) | live only |
| Bitget | 32K BTC | none | live only |
| OKX | 29K BTC | 5 min bars about two days, hourly eight days | live only (the history sends no CORS header) |
| Deribit | 10K BTC (USD at the mark) | none | live only |

Binance, the series shown before, is 36 % of the 261K BTC these six hold.

## How they are added

Per venue, in coins (Deribit's USD divided by its mark when read, OKX's `oiCcy` not its contracts). A bar's sum counts every venue with a
bar there, and a venue read every five minutes (or one reading missed) is carried at its last reading for up to two timeframes or ten
minutes, whichever is longer: open interest is a level. A bar's change is the sum of each venue's own change from the bar one timeframe
earlier, over the venues present at both: a venue whose readings start (Bitget and Deribit when sampling starts, any venue after a gap)
would otherwise show its whole open interest as opened in one bar. NL/NS gives each venue's change to its own takers and then adds the
venues up: one venue's buyers say nothing about who opened positions on another.

## Cost

- Server: four more REST readings a minute (`OiSampler`), and at start Bybit's and OKX's history (two to three requests each), written to
  the history store a few hundred rows a turn. `oi_bars` keeps a row per instrument and minute for 30 days: six venues are about 260 000
  rows, some 55 MB, inside the store's 224 MiB budget (the file was 14 MB). A spare-port server with the 20 order-book venues of the saved
  selection used 45 to 60 % of a core with this, as before it, and held 576 five-minute bars for Bybit and OKX after three minutes.
- Page: six small OI requests instead of one every 20 s, and six live readings a minute while the page is open (browser mode).

## Not done

- Hourly OKX history before its two days of five-minute bars is read by the server only.
- A venue switched off in the Venues chips is left out of the sum from then on, back through its whole history: the sum is of the
  venues on now, not of those that were on then.

## NL/NS read per market (second pass, same day)

Three flaws in the first NL/NS over venues, found reviewing it before building on it:

- **Deribit's inverse perpetual.** Its open interest is a count of USD contracts, read as coins at each reading's mark. A 2 % move of the
  price with no contract opened or closed read as 2 % of its open interest opened or closed, and NL/NS gave that to the takers. The hub now
  puts an inverse market's bars in coins at one price, the newest close on screen (`atOnePrice`), before the sum, the pane or NL/NS see them.
- **Sparse readings.** A venue read every five minutes, or hourly in older history, changes once, by the whole step, in the candle of its
  new reading. That step was given to the takers of that one candle. It is now given to the takers of every candle since the reading before
  (`nlnsModel` sums the market's delta over them), and the candle is marked coarse when that stretch is longer than a candle.
- **Mixed candles.** Each venue was turned into one signed number before the venues were added, so new longs on one venue and new shorts on
  another summed to 0 and read "no change". Each market is now classified into the four parts first; a candle whose parts pull both ways
  reads "Both ways" and the popup lists them, with the longs' and shorts' own sums beside the net.

Recorded flow that came out even (takers even) is no longer handed to the price's way: only flow that was not recorded is.
