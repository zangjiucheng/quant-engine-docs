# quant-engine docs

C++ backtesting and factor research, a headless IBKR trading daemon,
and **QE Fleet** — the desktop app you watch and deploy them from.
Choose a workflow below; use Reference for syntax and output formats.

## Start here

1. [Download for macOS](download.md) and complete [first-time setup](install.md).
2. Run [your first factor](first-factor-tutorial.md), then use the
   [QE Fleet walkthrough](fleet-walkthrough.md) to explore the screens.
3. Read [Safety and known limitations](safety-and-limitations.md) before
   interpreting results or deploying a strategy.

## Find a workflow

| Task | Start with | Details |
|---|---|---|
| Write and run a strategy | [`.qe` language](qe-language.md) | [Signal expressions](dsl-grammar.md), [result schema](results-json.md) |
| Research a factor | [Factor research](factor-research.md) | [Forecasting](forecasting.md) |
| Validate and combine strategies | [Walk-forward validation](walk-forward.md) | [Multi-strategy portfolios](multi-strategy.md) |
| Connect a broker | [IBKR connectivity](ibkr-connectivity.md) | [Credentials](broker-credentials.md) |
| Deploy a daemon | [Safety model](live-trading-safety.md), then [runbook](live-trading-runbook.md) | [Configuration and accounting](daemon-configuration.md) |
| Verify paper trading | [Paper-trading verification](ibkr-paper-verification.md) | [Daemon smoke checklist](qe-daemon-smoke.md) |
| Use Python | [Python client and bindings](python.md) | [Result schema](results-json.md) |
| Understand the implementation | [Architecture](architecture.md) | [QE Fleet walkthrough](fleet-walkthrough.md) |
| Upgrade an existing deployment | [Upgrade procedure](live-trading-runbook.md#upgrading-an-existing-deployment) | [Migration and incident history](release-history.md) |

For options formulas and conventions, see [Options pricing](options-model.md).
C++ API documentation is generated separately with Doxygen.

## Internal testing access

Engine and application source is currently private. For
internal-testing access, contact **jiucheng.zang@proton.me**.

## Maintaining these docs

See [Documentation maintenance](documentation-maintenance.md) for page ownership,
historical evidence, link checks and the publishing boundary.
