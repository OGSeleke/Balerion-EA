# Balerion – The Black Dread

> *"Fire made flesh. The oldest dread. The last word in gold."*

**Balerion** is a MetaTrader 5 Expert Advisor built for **XAUUSD on the M5 timeframe**. It is not a simple signal EA — it is a multi-layer decision engine that evaluates market structure, regime, bias, session context, and volatility before allowing a single trade to open, then actively manages the basket through tiered trailing, deep recovery, and session-aware protection.

Version: **3.0** | Author: **Oageng Seleke** | Magic: `20260417`

---

## Table of Contents

- [Overview](#overview)
- [How It Works — The Decision Engine](#how-it-works--the-decision-engine)
- [V3 Brain Modules](#v3-brain-modules)
- [Basket and Trade Management](#basket-and-trade-management)
- [Protection Layers](#protection-layers)
- [Session and Time Controls](#session-and-time-controls)
- [Entry Filters](#entry-filters)
- [HUD — Live Visual Monitor](#hud--live-visual-monitor)
- [Input Parameters Reference](#input-parameters-reference)
- [Architecture Summary](#architecture-summary)
- [Screenshots](#screenshots)
- [Setup Notes](#setup-notes)
- [Disclaimer](#disclaimer)

---

## Overview

Balerion treats every entry as a question, not a certainty. A trigger signal must pass through the **V3 Brain** — a scoring engine that weighs directional bias, market regime, trend exhaustion, higher-timeframe structure, session timing, tick volume, and macro/news context before deciding whether to allow, tighten, or block the trade.

Once a basket is open, management shifts to a **depth-aware protection stack**: a single trade gets one set of rules, a 2-leg basket gets another, and a 7–9 leg recovery zone gets its own dedicated protector. The system is designed to behave differently at each depth level — not because the code is complicated, but because the problem is.

---

## How It Works — The Decision Engine

Every M5 bar, Balerion runs the following sequence:

```
[M5 Trigger fires]
       ↓
[Entry Judge called]
       ↓
  ┌────────────────────────────────────────┐
  │ 1. Bias Engine       (DXY, Yields,     │
  │                       Internal EMA,    │
  │                       FastM5 bars,     │
  │                       Macro file)      │
  │ 2. Trend Dying       (M5 / M15 / H1)  │
  │ 3. Regime Engine     (Range / Trend /  │
  │                       Breakout / Chaos)│
  │ 4. Session Context   (Quiet / Active / │
  │                       Late / Trans.)   │
  │ 5. Tick Volume Judge (Normal / Expanding│
  │                       / Exhausting)    │
  │ 6. HTF Confluence    (H1 / H4 / Both)  │
  │ 7. Trend Align Gate  (EMA 50/200 H1)  │
  │ 8. News Quality      (None/Low/Med/High)│
  └────────────────────────────────────────┘
       ↓
  [Score assembled]
       ↓
  BLOCK / ALLOW_STRICT / ALLOW
```

The final `ENTRY_DECISION` state controls whether:
- A first trade can open at all
- The second leg gets a tighter pip-step (strict mode adds ~25%)
- The entry is hard-blocked regardless of score

---

## V3 Brain Modules

### Bias Engine
Aggregates directional opinion from multiple sources and produces a weighted `BIAS_BULL`, `BIAS_NEUTRAL`, or `BIAS_BEAR` score. Sources include:

| Source | Weight (default) |
|--------|-----------------|
| DXY vote (from macro file) | 3 |
| Real yields vote (from macro file) | 3 |
| News quality vote | 1 |
| Internal EMA trend (H1 + H4) | 1 |
| Fast M5 momentum bars | 2 |

Macro data is read from an external `balerion_macro_news.json` file (written by a companion Python feeder). If the file is older than 10 minutes, Balerion marks it stale and clears all macro votes automatically.

### Trend Dying Engine
Detects when an existing trend is losing energy across M5, M15, and H1. Produces a state score from `TD_NONE` → `TD_MICRO` → `TD_MAIN_WEAK` → `TD_MAIN_CONFIRMED` → `TD_STRONG`.

A dying trend is a **positive** for Balerion's counter-trend fades. The stronger the dying signal, the more it can rescue a trade that would otherwise have a low score.

### Regime Engine
Classifies current market condition as one of:

| Regime | Description |
|--------|-------------|
| `RANGE` | Price oscillating within ATR bounds |
| `TREND_UP` | Fast bullish impulse detected |
| `TREND_DOWN` | Fast bearish impulse detected |
| `BREAKOUT` | Price breaking session structure |
| `CHAOTIC` | Micro-spike emergency detected — hard block |

Regime determines the minimum score threshold for entry (`MinScoreRange`, `MinScoreTrend`, `MinScoreBreakout`).

### HTF Confluence Engine
Checks whether current price is near a higher-timeframe structural level (swing high or swing low). Proximity within `HTFConfluenceZonePts` earns bonus score points:

- Near H1 swing: `+2`
- Near H4 swing: `+4`
- Near both H1 and H4: `+6`

H4 confluence can optionally be made mandatory via `RequireH4Confluence`.

### Session Context Engine
Classifies current SAST time as `QUIET`, `NORMAL`, `ACTIVE`, `LATE`, or `TRANSITION` based on the hour and a comparison between current ATR and recent average ATR. Active sessions with expanding volatility add score points; quiet sessions with compressed volatility subtract them.

### Tick Volume Judge
Compares current M5 tick volume against the `TickVolLookback` average. If volume is expanding significantly, it scores negatively (momentum chase risk). If volume is exhausting, it scores positively (energy running out = better fade opportunity).

### Entry Judge Score Assembly
The judge adds all component scores together and compares to a regime-based threshold. The final decision:

- `score >= needed + 2` → **ALLOW** (full first entry)
- `score >= needed` → **ALLOW_STRICT** (entry allowed but second leg pip-step widened)
- `score < needed` → **BLOCK**

A `strongRescue` flag (active when H4 confluence or strong dying trend exists) lowers the threshold by 2 points. The `BrainScoreGlobalOffset` input acts as a global dial: positive values tighten all thresholds, negative values loosen them.

---

## Basket and Trade Management

### Lot Progression
Balerion uses a tiered lot model across up to 9 legs:

| Zone | Legs | Lot Multiplier | PipStep Multiplier |
|------|------|---------------|-------------------|
| Tier 1 | 1–3 | 1× base lot | 1.0× |
| Tier 2 | 4–6 | 2× base lot | `Tier2GapMult` (default 1.5×) |
| Tier 3 | 7–9 | 3× base lot | `Tier3GapMult` (default 2.0×) |

### Average-Price TP
Take profit is calculated from the **basket average price**, not from individual entry prices. The TP distance is `TakeProfitPts` for legs 1–3. For legs 4–6, TP widens to `TakeProfitPts × DeepTPMultiplier` (default 5×) to give the deep basket room to recover.

### Adaptive PipStep
Each new leg's distance from the last entry respects both a base `PipStep` and an ATR floor. If `ATRPipStepPct > 0`, the minimum pip step is `ATR(14) × ATRPipStepPct`, ensuring the grid stays relevant to current volatility. During active exhaustion spikes, `SpikeMultiplier` widens the step further.

### CHoCH Reclaim Gate
When price makes an adverse spike large enough to register, Balerion blocks new grid legs until price **reclaims** the spike level (closes back beyond it by `CHoCHReclaimBufferPts`). This prevents automatically adding into confirmed breakouts.

### Session Breakout Guard
If price is beyond the session high/low (measured over `SessionLookbackBars` M5 bars), the pip-step multiplier increases for new adds, making the grid more conservative when price is already in extended territory.

---

## Protection Layers

### Single Trade Trailing Stop (Legs = 1)
Two-stage stepped trail:

| Stage | Arms at | SL Locks to | Trail Distance |
|-------|---------|------------|---------------|
| Stage 1 | `TrailArm1Pts` profit | Entry + `TrailLock1Pts` | `TrailDist1Pts` behind price |
| Stage 2 | `TrailArm2Pts` profit | Upgrades automatically | `TrailDist2Pts` behind price |

### Basket Trail (Legs 2–3)
Separate USD-based arming for the basket as a whole. Arms when floating profit reaches `Leg2TrailArmUSD` or `Leg3TrailArmUSD`, then trails the basket SL behind price by the configured distance.

### Deep Basket Trail (Legs 4–6)
Per-leg USD arming with a **lock floor** — once armed, the SL can never move below a minimum guaranteed profit level. Each leg profile has its own arm threshold, lock USD, and trail distance:

| Leg | Arm (USD) | Lock Floor (USD) | Trail Distance (pts) |
|-----|-----------|-----------------|----------------------|
| 4 | 80 | 60 | 1000 |
| 5 | 120 | 70 | 1000 |
| 6 | 170 | 100 | 1200 |

### Tier 3 Recovery Protector (Legs 7–9)
At the deepest recovery zone, a dedicated protector activates. It:
- Compresses the TP closer to current price (scaled by `Leg7/8/9TPMultiplier`)
- Arms a recovery lock SL based on USD profit progress toward TP
- Monitors stall bars and ATR-relative struggle
- Locks a profit floor once sufficient progress has been made
- Triggers early exit if price stalls for too many bars without making progress

### Late-Session Compression SL
After `LateSessionSLHour:LateSessionSLMinute` (default 22:55 SAST), Balerion checks if the market is compressed (range < `CompressionRatio × ATR` over `CompressionBars` bars). If compressed, it locks a stop loss at `CompressionSLPts` from the average basket price to protect the overnight basket.

### Micro-Volatility Emergency Exit
Tracks all ticks over a `MicroWindowSec` window. If the range of that micro-window exceeds `MicroEmergencyPts` **and** the basket loss is within `MicroMaxLossForEmergency`, Balerion can emergency-exit the basket. The adverse ratio check (`MicroAdverseRatio`) ensures the spike must actually be moving against the basket — not just general volatility.

### Basket Hard Stop
`MaxBasketLoss` (USD) acts as a global basket hard stop. If total floating loss reaches this level, all positions are closed regardless of any other trailing or protection state.

---

## Session and Time Controls

| Control | Default | Description |
|---------|---------|-------------|
| Monday Filter | 1:00 AM SAST | No trading before this hour on Mondays |
| Friday No New Entries | 7:00 PM SAST | No first entries, existing basket only |
| Friday Basket Freeze | 8:00 PM SAST | No new grid levels added |
| Friday Force Close | 9:55 PM SAST | All positions closed |
| News Block Before | 30 minutes | Block entries this many minutes before HIGH-impact news |
| News Block After | 15 minutes | Block entries this many minutes after HIGH-impact news |
| Late Session SL | 22:55 SAST | Compression SL activated if market is tight |
| Spread Filter | 50 pts | Reject entries when spread exceeds this |
| Slippage Control | 80 pts | Maximum deviation allowed on order execution |

All times use SAST (UTC+2). The `BrokerOffsetFromSAST` input adjusts for brokers running on a different server timezone (default +1 for UTC+3 brokers).

---

## Entry Filters

Optional pre-brain filters that can be toggled independently:

| Filter | Input | Default |
|--------|-------|---------|
| RSI | `UseRSIFilter` | Off |
| MACD | `UseMACDFilter` | Off |
| Doji rejection | `UseDojiFilter` | Off |
| Session breakout guard | `UseBreakoutGuard` | On |
| HTF structural guard | `UseHTFGuard` | On |
| Trend alignment gate | `UseTrendAlignFilter` | On |

The **HTF Structural Guard** tracks how many consecutive bars have closed beyond a key H1 structural level. If this count reaches `HTFBreakConfirmBars`, the entry is penalised heavily. If no confluence or dying trend offsets it, the entry is hard-blocked.

The **Trend Alignment Gate** scores the gap between EMA50 and EMA200 on H1. A large counter-trend gap applies a score penalty (`-1` for strong, `-2` for extreme). It no longer hard-blocks — it contributes to the score system, allowing strong confluence or dying trend to override it when warranted.

---

## HUD — Live Visual Monitor

Balerion includes a styled on-chart HUD panel that updates on every tick. The panel uses a dark charcoal background with a gold border and gold title, and shows 20 live data lines:

| Line | Information |
|------|------------|
| 1 | Status (Idle / Basket Active) |
| 2 | Symbol and current basket direction |
| 3 | Trade count (Buy / Sell breakdown) |
| 4 | Floating P&L and basket hard stop limit |
| 5 | Next grid level and next lot size |
| 6 | Current spread vs maximum allowed |
| 7 | ATR(14) in points and TP distance |
| 8 | PipStep, ATR floor, and adaptive step |
| 9 | Micro-volatility window range |
| 10 | Single trail state + basket trail state |
| 11 | Deep basket trail state + late session SL state |
| 12 | Tier 3 recovery state |
| 13 | CHoCH gate status + HTF structural break count |
| 14 | Trend alignment gate score and threshold |
| 15 | Bias state + Regime state |
| 16 | Macro bias + macro confidence |
| 17 | Macro file freshness + news quality |
| 18 | Next news event summary |
| 19 | Entry Decision (BLOCK / ALLOW / ALLOW_STRICT) + score |
| 20 | TA score + threshold + tick volume state |

Color coding uses white for normal, green for active/positive, orange for warnings, and red for danger/block states. The Balerion logo and title appear at the bottom of the panel.

Brain events (ALLOW, ALLOW_STRICT, BLOCK) are also plotted as text markers directly on the chart candles for visual review.

---

## Input Parameters Reference

### General Settings
| Parameter | Default | Description |
|-----------|---------|-------------|
| `MagicNumber` | 20260417 | Unique EA identifier |
| `Lots` | 0.01 | Base lot size |
| `MaxTrades` | 9 | Maximum open legs |
| `TakeProfitPts` | 500 | TP distance from average price (points) |

### Grid Settings
| Parameter | Default | Description |
|-----------|---------|-------------|
| `PipStep` | 250 | Base distance between grid levels (points) |
| `ATRPipStepPct` | 0.30 | ATR-based minimum PipStep (0 = off) |
| `MaxBasketLoss` | 350 | Basket hard stop in USD |
| `Tier2GapMult` | 1.5 | PipStep multiplier for legs 4–6 |
| `Tier3GapMult` | 2.0 | PipStep multiplier for legs 7–9 |

### Brain Score Tuning
| Parameter | Default | Description |
|-----------|---------|-------------|
| `MinScoreRange` | 5 | Minimum score to allow entry in ranging market |
| `MinScoreTrend` | 6 | Minimum score in trending market |
| `MinScoreBreakout` | 10 | Minimum score during active breakout |
| `BrainScoreGlobalOffset` | -1 | Global threshold dial (+N tightens, -N loosens) |

### Logging
| Parameter | Default | Description |
|-----------|---------|-------------|
| `LogBrainStateChanges` | true | Log bias/regime/entry state changes |
| `LogBrainEntryReasons` | true | Log full entry decision with score breakdown |
| `LogBrainTuningDetail` | true | Log detailed component values per bar |
| `LogBrainEveryBar` | false | Log every bar snapshot (verbose, for tuning only) |

---

## Architecture Summary

```
┌─────────────────────────────────────────────────────┐
│                    BALERION v3.0                    │
├──────────────┬──────────────────────────────────────┤
│  TRIGGER     │  Counter-trend signal on M5          │
├──────────────┼──────────────────────────────────────┤
│  V3 BRAIN    │  Bias + Regime + Dying + HTF +       │
│              │  Session + TickVol + News → SCORE    │
├──────────────┼──────────────────────────────────────┤
│  GRID        │  CHoCH gate + ATR adaptive step +    │
│              │  Exhaustion spike guard              │
├──────────────┼──────────────────────────────────────┤
│  MANAGEMENT  │  Single trail → Basket trail →       │
│              │  Deep basket trail → Tier3 protector │
├──────────────┼──────────────────────────────────────┤
│  SESSION     │  Monday / Friday / News / Late-SL /  │
│              │  Spread / Compression / Emergency    │
├──────────────┼──────────────────────────────────────┤
│  HUD         │  20-line live panel + chart events   │
└──────────────┴──────────────────────────────────────┘
```

---

## Screenshots

> *Suggested screenshots to add to this section:*

| Screenshot | What to show |
|-----------|-------------|
| `hud_idle.png` | HUD panel on a clean chart with no basket active |
| `hud_basket.png` | HUD panel with an active 3–4 leg basket |
| `hud_decision.png` | HUD showing ALLOW_STRICT or BLOCK decision state |
| `chart_events.png` | Chart view with brain event markers plotted on candles |
| `tier3_recovery.png` | HUD during a deep basket with Tier3 protector active |

**See the [Screenshots](#screenshots) section below once images are added.**

---

## Setup Notes

1. **Timeframe:** Attach to XAUUSD M5 only. The EA will refuse to initialize on any other timeframe.
2. **Macro file:** The bias engine reads `balerion_macro_news.json` from the MT5 Common Files folder. A companion Python script should write this file every few minutes. Without it, DXY, Yields, and News votes are zero and the file shows as `MISS` or `STALE` on the HUD.
3. **Logo resource:** The HUD uses a `balerion_logo.bmp` image embedded in the EA. Ensure the `Images` subfolder is present in the EA project when compiling.
4. **Broker offset:** Set `BrokerOffsetFromSAST` to match your broker's server time relative to SAST (UTC+2). Most UTC+3 brokers use `+1`.
5. **Starting safe:** `BrainScoreGlobalOffset = -1` slightly loosens thresholds. Set it to `0` or `+1` for tighter filtering. Never set it below `-3` in live trading.

---

## Disclaimer

This repository is a **project showcase only**. It documents the architecture, features, and development direction of Balerion. The full proprietary source code is not released.

This is not financial advice. Past performance of any EA does not guarantee future results. Trading XAUUSD involves significant risk of capital loss. Use at your own risk and always test thoroughly in a demo environment before any live deployment.

---

*Balerion – The Black Dread. Built for gold. Built to last.*
