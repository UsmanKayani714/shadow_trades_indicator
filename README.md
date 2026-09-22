# Shadow Trades

A Pine Script v6 TradingView indicator that reads **higher-timeframe (HTF) market
structure** and projects it onto whatever chart you are actually trading from.

It draws five things, all derived from closed HTF candles only:

| Object | What it is |
| --- | --- |
| **MSB / MSS** | A market structure **break** (continuation) or **shift** (bias flip), marked at the swing that gave way. |
| **Order Blocks** | The last opposite-direction HTF candle before a structure break. |
| **Breakers** | A failed order block — one price closed through — re-armed by an opposite structure break. |
| **Fair Value Gaps** | Three-candle HTF imbalance (`low[0] > high[2]`, or `high[0] < low[2]`). |
| **Dealing range** | The live HTF range: protected swing on one side, running extreme on the other, plus equilibrium (50%). |

Everything is placed with **bar-time coordinates** (`xloc.bar_time`), so a 4H
order block sits on the correct candles whether you are looking at a 4H, 15m or
1m chart.

---

## Install

1. TradingView → **Pine Editor** → **Open → New indicator**.
2. Replace the contents with [`shadow_trades.pine`](shadow_trades.pine).
3. **Save**, then **Add to chart**.

No external dependencies.

---

## How it works

### Structure

Swing pivots are confirmed on the HTF: a candle is a pivot high when `pvLen`
HTF candles either side have strictly lower highs (and the mirror for pivot
lows). Confirmation is therefore intentionally `pvLen` HTF bars late — that is
what makes the pivot final rather than provisional.

A **break** requires an HTF *close* through the pivot level, not a wick through
it. A break that agrees with the current bias prints **MSB**; one that reverses
it prints **MSS**.

Because a pivot is only valid when no neighbouring candle exceeded it, a break
can never be "missed" during the confirmation window — the first close through
the level always happens after the pivot is armed.

### Order blocks and breakers

On a bullish break, the script walks back from the candle *before* the break
looking for the most recent down-close candle; that candle becomes the bullish
order block (wick to wick by default). The mirror applies to bearish breaks.

An order block is **invalidated** when an HTF candle *closes* through its far
side — below the low of a bullish OB, above the high of a bearish OB. An
invalidated order block is not discarded: the next opposite structure break
promotes it to a **breaker**, which is itself invalidated when price closes back
through it.

### Fair value gaps

Checked on the three most recent closed HTF candles. Gaps shrink as price trades
into them (`fvgShrink`) and are retired once fully filled. Wick-fill is the
default (`fvgByWick`); flip it to close-fill in the constants block.

### Dealing range

On each break the protected swing on the origin side is fixed and the opposite
side tracks the expansion, so the range always describes the leg currently in
play. Equilibrium is the midpoint.

---

## Non-repainting design

HTF data is pulled with the standard confirmed-bar idiom:

```pine
request.security(syminfo.tickerid, htf, [open[1], high[1], ...], lookahead = barmerge.lookahead_on)
```

`lookahead_on` combined with the `[1]` offset returns the **last closed** HTF
bar, aligned identically on historical and realtime bars. Nothing is drawn from
a candle that has not finished, and nothing moves once drawn (other than an FVG
shrinking as it is filled, which is by design).

This idiom is only safe when the requested resolution is **at or above** the
chart's. If you select an HTF below the chart timeframe, the script detects it,
falls back to the chart timeframe, and prints a warning label on the last bar
rather than quietly serving you lookahead bias.

---

## Settings

### Basic

| Setting | Default | Notes |
| --- | --- | --- |
| Higher Timeframe (HTF) Structure | `Chart` | `Chart` follows the chart's own resolution. Selecting a resolution below the chart's falls back to the chart timeframe. |
| Hide Historical | on | On: only POIs and structure from the current dealing range survive; a POI is deleted the moment an HTF candle closes through it. Off: everything stays, invalidated POIs freeze and dim. |
| Display HTF Dealing Range | on | |
| Display Range 50% Line | on | |
| Visual Theme | `Terminal` | Sets the structure accent and fill opacity. Zone colours come from **Style Options**. |
| Label Active POIs | on | Prints the HTF tag and zone name inside each box. |
| POI Text Offset (bars) | 16 | How far past the last candle the POI label sits. |

### Points of interest

Three slots, each assignable to `Order Blocks`, `Breakers`, `Fair Value Gaps` or
`None`, with a per-slot **Extend** toggle (run to the right edge, or stop at the
current bar). `Max POIs per type / side` caps how many of each kind are kept.

Order blocks are tracked internally even when not displayed, because breakers
are derived from them.

### Structure

| Setting | Default | Notes |
| --- | --- | --- |
| Show MSS / MSB | on | |
| Swing lookback (HTF bars) | 2 | Candles required either side of a swing. Higher = fewer, larger swings. |
| MSS / MSB kept when Hide Historical is on | 2 | |

### Style

Per-object colours for range lines, order blocks, breakers and FVGs, plus line
thickness and an optional 50/200 EMA pair (chart timeframe).

### Fixed behaviour

A block of constants near the top of the script, deliberately kept out of the
settings panel:

| Constant | Default | Meaning |
| --- | --- | --- |
| `rngDim` | 45 | Transparency of the dealing-range lines. |
| `useWick` | `true` | Order block spans the full candle, wick to wick. |
| `obLook` | 30 | HTF bars searched back for the order block candle. |
| `msLen` | 4 | Length of an MSS / MSB marker, in HTF bars. |
| `fvgMinAtr` | 0.0 | Minimum gap size in HTF ATR14 multiples (0 = no filter). |
| `fvgShrink` | `true` | Trim a gap as price fills into it. |
| `fvgByWick` | `true` | A gap fills on wicks rather than closes. |
| `regCap` | 300 | POI registry depth; bounds the per-bar invalidation scans. |
| `emaLenF` / `emaLenS` | 50 / 200 | EMA lengths. |

---

## Alerts

Two `alertcondition` entries are exposed:

- **HTF Bullish Structure (MSB / MSS)**
- **HTF Bearish Structure (MSB / MSS)**

Both fire once, on the chart bar where the HTF candle that caused the break
rolls over.

---

## Review notes

### Fixed in this version

1. **Lookahead bias when the selected HTF was below the chart's.**
   `lookahead_on` on a lower-resolution request hands historical bars data that
   had not printed yet — e.g. a 1H chart with "30 minutes" selected would see
   the second half of the hour before it happened. Now detected up front; the
   script falls back to the chart timeframe and labels the chart to say so.

2. **Unbounded registries.** The order block and FVG registries grew for the
   life of the chart, and the invalidation scans in steps 1a/1c walked every
   entry ever recorded on every HTF bar. On a long history at a fast HTF that is
   quadratic work and eventually trips Pine's per-bar execution limit. Both
   registries are now capped at `regCap` entries, and drawings hold an
   **absolute registry id** instead of a slot index so trimming the front can
   never re-point a box at the wrong zone.

3. **Order block could be the breaking candle itself.** The search started at
   the break candle, so a candle that gapped up, sold off, and still closed
   through the swing became its own order block. The search now starts one
   candle earlier, which is what "the candle preceding the break" means.

4. **Range could go missing on the first break.** If the first structure break
   arrived before any opposite pivot had been confirmed, the protected side was
   `na` and the whole dealing range — lines and equilibrium — silently failed to
   draw until the next break. The breaking candle's extreme now stands in.

5. **`prune` evicted live POIs ahead of dead ones.** With *Hide Historical* off,
   frozen invalidated boxes counted against `Max POIs per type / side` and, being
   oldest, an active zone could be pushed off the chart while a finished one
   stayed. Pruning now prefers the oldest invalidated zone and only falls back to
   a live one when none is available.

6. **Half-formed HTF candles could enter the store.** `newBar` only checked the
   timestamp. During warm-up the timestamp can arrive before the OHLC, pushing
   `na` prices into the arrays and producing a bogus pivot. The open is now
   checked alongside it.

7. **Per-bar right-edge refresh.** `refresh()` rewrote every box's right edge on
   every chart bar, although only the last bar can change it. It now runs once,
   under `barstate.islast`. Identical output, materially cheaper.

8. **Tooltip wording.** The *Hide Historical* tooltip promised deletion "the
   moment price mitigates it", but the code requires an HTF *close* through the
   far side — invalidation, not mitigation-on-touch. The tooltip now matches the
   behaviour.

### Known limitations, left as-is

- **Invalidation vs. mitigation.** A zone is retired on a close through its far
  side, not on first touch. That is a defensible model — it keeps a zone alive
  through a wick raid — but it is a choice, and traders who want touch-based
  mitigation would need a second mode.
- **Protected swing vs. running extreme.** On a bias flip the range boundary is
  set to the last *confirmed* pivot. When the final push made its extreme within
  `pvLen` candles of the flip, no pivot was ever confirmed there, so the range
  understates the leg by that much. Taking `math.max(svPH, rngH)` would track the
  true extreme, at the cost of the "protected swing" semantics.
- **First break prints MSB.** With `bias == 0` there is no prior bias to shift
  from, so the very first break on a chart is labelled MSB. Arguably it should be
  unlabelled.
- **`poiOff` is measured in bar-time, not session time.** The right-edge
  projection is `time + chartTF × poiOff`, which runs through weekends and
  session gaps, so the POI label can sit further right than `poiOff` visible
  candles on a non-24/7 symbol. Cosmetic only.
- **Purged registry entries can still spawn breakers.** With *Hide Historical*
  on, `purgeOlder` removes drawings but leaves registry entries, so an order
  block from before the current range can still be promoted to a breaker later.
  This is arguably the point of a breaker, so it is left alone — but it does mean
  a box can appear from outside the "current range only" window.
