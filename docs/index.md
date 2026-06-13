# quant-engine docs

A high-performance C++ quant backtest + research engine and a
terminal-style native dashboard. This site is the user-facing
reference; C++ API docs are generated separately by Doxygen.

## Where to start

- **New here?** Walk through
  [Your first factor — a 10-minute tour](first-factor-tutorial.md).
  Zero terminal, just the dashboard.
- **First time launching the dashboard?**
  [Dashboard walkthrough](dashboard-walkthrough.md) covers the
  screens, the keymap, and the cold-start tour.
- **Going live?** Read the
  [safety model](live-trading-safety.md) first, then
  [IBKR connectivity](ibkr-connectivity.md) and the
  [paper-trading verification](ibkr-paper-verification.md)
  before flipping `broker = "ibkr-live"`.

## I want to…

| Task                                        | Page                                                |
|---------------------------------------------|-----------------------------------------------------|
| **Write** a backtest / sweep / walk-forward | [`.qe` language](qe-language.md)                    |
| Write the per-bar **signal expression**     | [Signal DSL grammar](dsl-grammar.md)                |
| **Validate** a strategy IS / OOS            | [Walk-forward validation](walk-forward.md)          |
| **Combine** several signals into a book     | [Multi-strategy portfolios](multi-strategy.md)      |
| **Forecast** returns with rolling ridge     | [Forecasting](forecasting.md)                       |
| **Screen** factors over a cross-section     | [Factor research](factor-research.md)               |
| **Price** options + see Greeks              | [Options pricing](options-model.md)                 |
| **Connect** to Interactive Brokers          | [IBKR connectivity](ibkr-connectivity.md)           |
| **Verify** a paper account end-to-end       | [Paper-trading verification](ibkr-paper-verification.md) |
| Understand the **safety net**               | [Safety model](live-trading-safety.md)              |
| Decode a TWS socket frame                   | [TWS wire protocol](ibkr-tws-protocol.md)           |

## Internal testing access

Source for the engine and dashboard is currently private. For
internal-testing access, contact **jiucheng.zang@proton.me**.
