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
it prints **MSS**. The very first break on a chart has no prior bias to continue,
so it establishes one and prints **MSS**.

Because a pivot is only valid when no neighbouring candle exceeded it, a break
can never be "missed" during the confirmation window — the first close through
the level always happens after the pivot is armed.

### Order blocks and breakers

Which candle becomes the order block is set by *Order block candle*:

- **Last candle** (default) — the most recent down-close candle before the
  break. That is the final pause before the impulse, so the zone sits close to
  price and is reachable on a shallow pullback.
- **Leg origin** — the down-close candle that reaches *deepest* into the leg,
  searched back as far as the swing the leg is built on. That is where the move
  began, so the zone sits lower, further from price, and survives much longer.

Either way the zone spans the candle wick to wick by default, and the mirror
applies to bearish breaks.

`Leg origin` searches back to the **leg anchor** — the swing the current bias
sequence is built on, set when the bias last *shifted* and held through every
continuation break after it. It is deliberately not the protected swing, which
ratchets up to each new higher low and would collapse the search to a candle or
two in a running trend.

Note that `Swing lookback` has little influence on either rule. The search is
anchored to the **break candle**, not to the pivot, so when one impulse candle
clears both a tight and a wide swing high the break lands on the same candle at
every lookback. The lookback changes which level is marked broken and how often
breaks fire — not, in most cases, where the block lands.

An order block is **retired** on whichever event *Retire an OB / Breaker on*
names: an HTF candle **closing through** its far side (the default — a wick raid
leaves the zone alive), or the **first touch** of the zone by any HTF candle.
Breakers follow the same setting.

Promotion is tracked separately and always needs a close through, whichever mode
you pick, because a close through the block is what makes it a *failed* one. So a
block retired on first touch can still be promoted to a **breaker** later, when
the next opposite structure break arrives.

With *Hide Historical* on, only blocks from within the current dealing range are
eligible for promotion, so a breaker cannot appear out of a leg you have already
told the script to forget.

### Fair value gaps

Checked on the three most recent closed HTF candles. Gaps shrink as price trades
into them (`fvgShrink`) and are retired once fully filled. Wick-fill is the
default (`fvgByWick`); flip it to close-fill in the constants block.

### Dealing range

On each break the boundary on the origin side is fixed and the opposite side
tracks the expansion, so the range always describes the leg currently in play.
Equilibrium is the midpoint.

Which boundary gets fixed depends on the break:

- On a **shift**, the leg that just ended owns it, and that leg's true extreme
  often prints inside the last `pvLen` candles where no pivot could ever be
  confirmed — so the running extreme is kept.
- On a **continuation**, the newest confirmed swing is the one being protected,
  so that is used instead.

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

`Order block candle` chooses between `Last candle` (default) and `Leg origin`,
described under **Order blocks and breakers** above.

`Retire an OB / Breaker on` chooses between `Close through` (default) and
`First touch`. Fair value gaps are unaffected — they keep their own fill model,
set by `fvgByWick` and `fvgShrink` in the constants block.

Order blocks are tracked internally even when not displayed, because breakers
are derived from them.

### Structure

| Setting | Default | Notes |
| --- | --- | --- |
| Show MSS / MSB | on | |
| Show diagnostics table | off | Prints what the last break actually decided: leg anchor, search window size, which candle was chosen. |
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
| `obLook` | 60 | HTF bars searched back for the order block candle. |
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

### Round 1 — correctness and cost

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

### Round 2 — behaviour

9. **Retirement was hard-wired to close-through.** A zone survived until an HTF
   candle closed past its far side, with no way to ask for touch-based
   mitigation. Now a setting. The awkward part is that breaker promotion *must*
   stay close-based — a block is only breaker material once price has closed
   through it — so the registry now carries two separate flags: `obBrk` (closed
   through, structural, drives promotion) and `obMit` (drawing retired, follows
   the setting). Collapsing them would have made `First touch` quietly kill
   every breaker.

10. **The dealing range understated the leg on a bias flip.** The boundary was
    set to the last *confirmed* pivot, but a leg's extreme routinely prints
    within `pvLen` candles of the flip, where no pivot can ever be confirmed —
    so the range stopped short of the actual high or low. On a shift the running
    extreme is now kept; on a continuation the newest swing still wins, which is
    the correct answer for that case.

11. **The first break on a chart printed MSB.** `isShift` tested `bias == -1`,
    so `bias == 0` fell through to "continuation" — of nothing. Now `bias != 1`,
    and the first break prints MSS.

12. **`POI Text Offset` did not do what it said.** Two problems at once. The
    name was box text with `text_halign = center` inside an extended box, so it
    landed halfway between the zone's own left edge and the projection point —
    a zone from 200 candles back put its name ~100 candles back, not 16 forward.
    And the projection itself was `time + chartTF × poiOff`, a wall-clock span,
    which weekends and session breaks stretch on anything that is not 24/7.
    The name is now a separate `xloc.bar_index` label pinned at
    `bar_index + poiOff` — a real bar count, measured from the right edge. The
    labels are pooled and repositioned on the last bar rather than recreated.

13. **`Hide Historical` leaked zones from outside the current range.**
    `purgeOlder` removed drawings but left registry entries, so an order block
    from a leg the user had told the script to forget could still be promoted
    into view as a breaker. Promotion is now gated on the block sitting inside
    the current range window.

### Round 3 — order block placement

14. **The order block candle is now selectable.** The one rule on offer was
    "last opposite-close candle before the break", which anchors the zone near
    the impulse rather than at its origin. `Leg origin` picks the candle that
    reaches deepest into the leg instead, bounded by the swing the leg is built
    on (`rngLt` / `rngHt`, already computed at that point). It falls back to the
    last-candle rule when a leg contains no opposite close inside its own
    bounds, so the option can never leave a break without a block.

### Round 4 — the leg anchor

15. **`Leg origin` searched the wrong window.** It was bounded by `rngLt`, the
    protected swing — but on a *continuation* break that is reassigned to the
    newest confirmed higher low, so in a running trend the window collapsed to a
    candle or two and the rule re-picked exactly what `Last candle` picks. The
    bound is now a separate leg anchor, set only when the bias shifts and held
    across every continuation after it. `obLook` also went 30 → 60 HTF bars, so
    a long leg is not clipped before its origin is reached.

16. **Added a diagnostics table.** Two rounds were spent guessing at which
    candle the script chose and why. It now reports that directly: leg anchor,
    search window in HTF bars, how far back the chosen candle sits from the
    break, and its range. Off by default.

### Behaviour worth knowing

- **`First touch` is aggressive by design.** An order block created a candle or
  two from price will often be touched almost immediately, because the candles
  that drove the break had to trade through the zone to get there. That is what
  touch mitigation means; it is why `Close through` is the default.
- **Fair value gaps ignore the retirement setting.** A gap has its own model —
  it shrinks as it is filled and retires once fully traversed, per `fvgShrink`
  and `fvgByWick`.
- **Pivot confirmation is `pvLen` HTF candles late, by construction.** That lag
  is what makes a pivot final rather than provisional. It cannot cause a missed
  break: pivot validity requires every neighbouring candle's high to sit below
  the pivot, so no candle inside the confirmation window can have closed through
  it.
