# MES + XSP Busby Pivot — Project Handoff
_Last updated: 2026-05-22_

---

## What this indicator does

A morning setup indicator for **0DTE XSP options** trading.  It maps three layered things onto the chart:

1. **Premarket box** — the range MMs defined overnight, colored by MES futures sentiment (green = bull, red = bear, yellow = neutral).
2. **Busby-style pivot levels** — (PM H + PM L + PM C) / 3 as a bias midpoint, with gamma proxy zones extending _beyond_ the box as ladder profit targets.
3. **Opening Range box** — the first N minutes of the regular session (the pullback/pop trap window before the continuation move).

### Trading strategy context
- Place order just before market open for a cheap 0DTE debit ($0.05–$0.10)
- **Bullish**: buy CALL at/near PM LOW strike → OTO STC at T1/T2 above PM HIGH
- **Bearish**: buy PUT at/near PM HIGH strike → OTO STC at T1/T2 below PM LOW
- Expecting a pullback (bull) or false pop (bear) into the OR, then continuation
- Ladder out at T1 and T2 rather than all-or-none exit
- Not held to close — catching the opening volatility spike only
- Apex used alongside for trend direction confirmation

---

## Level structure

```
T2↑  = PM_HIGH + range × lvl2Mult   ← bull ladder target 2
T1↑  = PM_HIGH + range × lvl1Mult   ← bull ladder target 1
PM HIGH ──────────────────────────── call breakout / put entry ref
Pivot = (PM H + PM L + PM C) / 3 ── bias midpoint (yellow line)
PM LOW ───────────────────────────── call entry ref / put breakdown
T1↓  = PM_LOW  − range × lvl1Mult   ← bear ladder target 1
T2↓  = PM_LOW  − range × lvl2Mult   ← bear ladder target 2
```

Default multipliers: 0.25 and 0.50 — adjust after a few sessions of observation.

---

## Key design decisions

### Why time-based session detection instead of `session.ispremarket`
`session.ispremarket` and `session.ismarket` are unreliable for CBOE derived indices like XSP — they silently return false on every bar, causing `var` locked values to freeze on the first historical bar's price (typically months-old data). Replaced with explicit NY-timezone math: `hour(time, "America/New_York") * 60 + minute(...)`.

### Why MES is used for premarket range, not price weighting
XSP (CBOE index) has no premarket bars on TradingView. MES futures (CME_MINI:ES1!) trade 23/7 and have data from 4:00 AM ET onward. The script accumulates MES 15-min H/L/C during 4:00–9:30 AM, then scales to XSP price units at market open using `close / mesClose` ratio. XSP premarket is tried first if available, MES second, prior day as last resort.

### Why MES is sentiment-only (not price-weighted into pivot)
ES1! trades at ~10× XSP's absolute price. Blending raw prices produced a pivot ~3,500–4,000 on a ~750 chart, destroying the Y-axis. MES is now used only to determine bull/bear/neutral box color via % change vs its own prior-day close.

### Strike snapping
Daily ATR determines the snap grid so levels land on real option strikes:
- ATR < 4 pt → 0.5 pt grid
- ATR 4–8 pt → 1.0 pt grid
- ATR 8–15 pt → 2.5 pt grid
- ATR > 15 pt → 5.0 pt grid

### Pine v5 compile gotcha
Multi-line ternary chains (`a ? b :\n c ? d : e`) cause silent compile failures in Pine v5. All multi-branch expressions must be on a single line or assigned to intermediate variables first. Every ternary in this script is single-line.

---

## Inputs / settings

| Setting | Default | Notes |
|---|---|---|
| MES / ES Symbol | CME_MINI:ES1! | Futures sentiment source |
| Snap levels to nearest option strike | On | Uses ATR-derived grid |
| MES neutral band (%) | 0.10% | ± threshold for yellow box |
| Level 1 multiplier | 0.25 | T1 extension beyond box edge |
| Level 2 multiplier | 0.50 | T2 extension beyond box edge |
| Show Opening Range box | On | Toggle |
| Opening Range (minutes) | 15 | Options: 5, 10, 15, 30 |
| Show open label + entry reference | On | 9:30 label with bias/targets |

---

## Known issues / watch list

- **MES premarket scaling**: The `close / mesClose` ratio is snapshotted at the 9:30 bar. On high-gap days this should still be accurate, but watch for days where the ratio looks off and the levels land at unexpected prices.
- **Prior-day fallback**: On days where the chart is loaded after 9:30 AM (no premarket accumulated), levels fall back to prior-day H/L/C. The open label will still show correct targets but they're based on yesterday's range, not today's premarket.
- **Multiplier tuning**: 0.25/0.50 are starting points. After a week of sessions, review whether T1/T2 are landing too tight or too wide relative to actual continuation moves.
- **OR box timing**: The OR box appears when the OR period _closes_ (not live during it). On 5-min charts with a 15-min OR, that means 3 bars in before the box appears.
- **Push to GitHub**: The sandbox cannot authenticate to HTTPS remotes. After each session run `git push` from Terminal in the TradingView folder.

---

## Files

| File | Description |
|---|---|
| `MES_XSP Busby Style Pivot Predictor.pine` | Main indicator — paste into TradingView Pine Editor |
| `MES_XSP Busby Pivot - HANDOFF.md` | This file |

## Repo
`https://github.com/majohnst/TradingView.git` — branch `main`

Last commit: `6c3d07b` — "Redesign Busby pivot: fix level anchoring, add MES premarket range, OR box, open label"

---

## Next iteration ideas (post live-test)
- Tune `tp1OptionPrice` / `tp2OptionPrice` based on observed OR-range sizes — verify Busby PM_HIGH ≈ OR_high + $1 holds on live sessions
- Collect data on bear-day spike-to-OR-high frequency vs bull-day pullback-to-OR-low frequency to confirm positive expectancy ratio
- Add optional alerts: price crossing Call TP1/TP2 or Put TP1/TP2 for manual trim notification
- Consider showing nearest 2–3 round-number strikes (5-pt handles) as static gamma reference lines

---

## v3 Bear Market Notes (future — do not build yet)

Current strategy is intentionally optimised for a bull-market regime.  Bear-day losses (~$20/day) are the "lotto ticket subscription cost" for staying in the game on the 4 bull days/week that generate $30–$330 returns.

When regime shifts to sustained downtrend, the following will need recalibration:

| Parameter | Bull market (now) | Bear market (v3) |
|---|---|---|
| `maxPremium` | $0.10 | $0.20–$0.30 (VIX elevated, spreads wider) |
| `tp1OptionPrice` | $1.00 | $1.50–$2.00 (larger required moves) |
| `sentThresh` | 0.10% | 0.20–0.30% (ES persistently negative) |
| `tradeDir` | Both | Bear Only (Puts) as interim step |
| Entry trigger | Bull: pullback to OR low (fires daily) | Bear: gap-down dead-cat-bounce to OR high (becomes the reliable signal) |
| OR range expectation | 1–3 pt normal | 5–10 pt on elevated VIX |

**Interim bear-regime setting change** (before v3 rebuild):
1. Flip `tradeDir` → `Bear Only (Puts)`
2. Raise `maxPremium` to `0.25`
3. Raise `tp1OptionPrice` to `1.50`, `tp2OptionPrice` to `3.00`
4. Widen `sentThresh` to `0.25%`

Rebuild as v3 only when those interim settings produce consistently mis-sized fills or the OR trigger logic visibly breaks down.
