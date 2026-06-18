# Balerion-EA

Balerion is a custom MetaTrader 5 expert advisor for XAUUSD built around structured trade filtering, basket management, adaptive risk controls, and a multi-layer decision engine. Rather than treating entries as simple signal triggers, the system evaluates regime, bias, volatility, higher-timeframe structure, session context, and trade quality before allowing execution.

## Overview

Balerion is designed for M5 execution on XAUUSD and combines counter-trend trigger logic with a broader decision framework intended to reduce poor entries and improve trade management. The project focuses not only on opening trades, but also on controlling how baskets evolve under changing market conditions through trailing logic, recovery protection, structural guards, and time-based controls.

This repository is a public showcase for the project. The full proprietary trading logic and complete source code are not published here.

## Core philosophy

The idea behind Balerion is that an entry signal alone is not enough. A valid setup should also survive context checks such as:

- What market regime is active.
- Whether broader directional bias supports or conflicts with the trade.
- Whether momentum is expanding or dying.
- Whether price is pressing into higher-timeframe structure.
- Whether session conditions, news windows, or late-session behaviour make the setup weaker.
- Whether the basket should be protected, frozen, compressed, or exited.

The result is a system built to behave more like a decision engine than a single-rule EA.

## Main capabilities

### 1. Signal and entry control
Balerion uses a trigger-based entry model and then passes that trigger through a layered decision engine. Before a first trade is allowed, the system can evaluate:

- Bias state
- Regime state
- Trend-dying conditions
- Tick volume behaviour
- Higher-timeframe confluence
- Trend alignment pressure
- News quality
- Session context

This produces a scored decision state such as `BLOCK`, `ALLOW`, or `ALLOW_STRICT`.

### 2. Basket management
Once a position exists, Balerion shifts from first-entry decision logic into basket management. It supports:

- Multi-leg basket growth
- Average-price based TP handling
- Basket hard stop control
- Leg-aware trailing behaviour
- Progressive management logic by basket depth

The EA is designed to treat a trade basket differently at 1 leg, 2–3 legs, 4–6 legs, and 7–9 legs.

### 3. Trailing and protection layers
Balerion includes several protection systems depending on basket depth:

- Single-trade stepped trailing stop
- Basket trail for legs 2–3
- Deep basket trail for legs 4–6
- Tier 3 recovery protector for legs 7–9
- Late-session compression SL
- Emergency micro-volatility exit logic

These layers are intended to adapt trade management as market conditions and basket size change.

### 4. Structural and context filters
The EA includes several controls to stop weak adds or dangerous continuation behaviour, including:

- CHoCH reclaim gate
- Session breakout guard
- HTF structural guard
- Monday start filter
- Friday session management
- News window blocking
- Trend alignment gate

This makes the EA more selective than a standard grid or trigger-only system.

### 5. Visual monitoring and diagnostics
Balerion includes a custom HUD and extensive logging so the live state of the system can be understood directly from the chart and terminal output. The HUD displays information such as:

- Basket status
- Direction and number of trades
- Floating P&L
- Spread and ATR
- Adaptive pip step
- Bias and regime state
- Macro/news freshness
- Entry decision state
- Tick volume state
- Trailing and basket protection state

The codebase also includes event plotting for important brain decisions and warning conditions.

## High-level architecture

At a high level, Balerion can be thought of as five major layers:

| Layer | Purpose |
|------|---------|
| Entry trigger | Detects initial directional opportunity |
| Decision engine | Scores whether the trigger should be allowed |
| Basket logic | Manages progression after the first trade |
| Protection systems | Trails, locks, compresses, or exits baskets |
| HUD and logs | Exposes live internal state for review |

This separation is one of the key ideas behind the project: entry, management, and protection are not treated as the same problem.

## Feature highlights

- XAUUSD M5 oriented design
- Scored first-entry decision engine
- Bias, regime, and trend-dying analysis
- Tick volume and session context filters
- Higher-timeframe confluence checks
- Adaptive pip-step handling
- CHoCH-based grid re-open logic
- Basket trail and deep basket trail logic
- Tier 3 recovery protection
- Friday and late-session controls
- External news and macro context integration
- Custom HUD and event plotting
- Detailed debug and tuning logs

## Trade management model

Balerion does not use a single fixed management style across all trade states. Instead, management evolves as exposure increases.

### Single trade
A single position can be handled by stepped trailing logic with stage-based arming and upgrade behaviour.

### Legs 2–3
Once the basket reaches 2 or 3 legs, the system can transition into basket trailing behaviour, managing the basket as a combined structure rather than a single isolated trade.

### Legs 4–6
At deeper exposure, a different trailing and protection profile can be used, including deep basket trail logic and adjusted TP behaviour.

### Legs 7–9
At the deepest recovery zone, the system can apply dedicated recovery protection logic intended to reduce the chance of uncontrolled basket deterioration.

## Risk and session awareness

Balerion includes multiple controls aimed at limiting poor behaviour during unstable or inconvenient market conditions. These include:

- Spread filtering
- Slippage control
- News window restrictions
- Monday delayed start
- Friday new-entry restrictions
- Friday basket freeze
- Friday forced close logic
- Late-session compression stop logic
- Micro-volatility emergency exit logic

These controls are intended to make the system more robust in live market conditions where execution quality and timing matter.

## Public repository scope

This repository is intended to document the project, its structure, and its development direction. It may include screenshots, selected notes, examples, or non-sensitive snippets, but it does not include the full proprietary EA source.

The goal of this public repo is to showcase the system professionally without exposing the complete implementation.

## Screenshots

Suggested additions for this section:

- Chart with HUD visible
- Example of first-entry decision states
- Basket management example
- Example of blocked vs allowed setups
- Log output showing bias, regime, and confluence state

## Development status

Current public status:

- Active private development
- Public showcase repository
- Full source withheld
- Documentation and presentation in progress

## Disclaimer

This repository is for project presentation and documentation purposes only. It is not financial advice, not a promise of performance, and not a public release of the full trading system.

## Contact / updates

Project updates, notes, and showcase material may be added over time as development continues.
