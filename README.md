# mt5-backtesting-lab

[![Rust CI](https://github.com/MT5-Automation-Labs/mt5-backtesting-lab/actions/workflows/rust.yml/badge.svg)](https://github.com/MT5-Automation-Labs/mt5-backtesting-lab/actions/workflows/rust.yml)
[![crates.io](https://img.shields.io/crates/v/mt5-quant.svg)](https://crates.io/crates/mt5-quant)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

MCP server for MetaTrader 5 strategy work on macOS and Linux. It compiles MQL5 Expert Advisors, runs the Strategy Tester through Wine, stores deals in SQLite, and exposes the result to an LLM client as tools.

Version 1.34.3 registers 92 tools. The process speaks MCP over stdio. `--port` opens a TCP listener for debugging the server itself.

Backtests are simulations of past prices. This project is not affiliated with MetaQuotes. See [License](#license).

## Requirements

- macOS or Linux
- A MetaTrader 5 terminal running under Wine or CrossOver
- Rust stable, when you build from source
- An MCP client that can launch a local stdio process

On Apple Silicon, CrossOver needs an `x86_64` Wine process. On a Linux host with no display, Wine needs Xvfb. Platform notes are in [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md).

## Install

From crates.io:

```bash
cargo install mt5-quant --locked
```

From a release archive:

```bash
# macOS Apple Silicon
curl -LO https://github.com/MT5-Automation-Labs/mt5-backtesting-lab/releases/latest/download/mcp-mt5-quant-macos-arm64.tar.gz
tar xzf mcp-mt5-quant-macos-arm64.tar.gz

# Linux x64
curl -LO https://github.com/MT5-Automation-Labs/mt5-backtesting-lab/releases/latest/download/mcp-mt5-quant-linux-x64.tar.gz
tar xzf mcp-mt5-quant-linux-x64.tar.gz
```

Put the `mt5-quant` binary on `PATH`.

From this repository:

```bash
cargo build --release
# binary: target/release/mt5-quant
```

## Configure

`scripts/setup.sh` detects Wine and the MetaTrader 5 terminal, then writes `config/mt5-quant.yaml`. That file is gitignored. A binary built from this tree reads it. A portable copy reads `config/mt5-quant.yaml` beside the executable when that file exists, and otherwise `~/.config/mt5-quant/config/mt5-quant.yaml`. `MT5_MCP_HOME` overrides both: the file is `$MT5_MCP_HOME/config/mt5-quant.yaml`.

```bash
bash scripts/setup.sh
```

The two required keys:

```yaml
wine_executable: "/path/to/wine64"
terminal_dir: "/path/to/MetaTrader 5"
```

The template is [config/mt5-quant.example.yaml](config/mt5-quant.example.yaml). Every key, and the lookup order, is in [docs/CONFIG.md](docs/CONFIG.md).

## Register with a client

The command is `mt5-quant` and the argument is `--stdio`.

Claude Code:

```bash
claude mcp add mt5-quant -- mt5-quant --stdio
```

Claude Desktop and Cursor use the same JSON. Cursor reads `.cursor/mcp.json`. Claude Desktop reads `~/Library/Application Support/Claude/claude_desktop_config.json`.

```json
{
  "mcpServers": {
    "mt5-quant": {
      "command": "mt5-quant",
      "args": ["--stdio"]
    }
  }
}
```

Codex CLI, `~/.codex/config.toml`:

```toml
[mcp_servers.mt5-quant]
command = "mt5-quant"
args = ["--stdio"]
```

opencode, `.opencode/opencode.jsonc`:

```jsonc
{
  "mcp": {
    "mt5-quant": {
      "type": "local",
      "command": ["mt5-quant", "--stdio"],
      "enabled": true
    }
  }
}
```

`mt5-quant --port 8765` listens on TCP while you debug the server. `verify_setup` is the first tool call after install.

## Run a backtest

Ask the client:

```text
Run a backtest on MyEA from 2025.01.01 to 2025.03.31
```

Dates in tool arguments are `YYYY.MM.DD`. The expert name has no path and no extension.

`run_backtest` stays open until the tester finishes. It runs five stages:

1. **Compile.** MetaEditor builds the `.mq5` through Wine. The terminal caches `.ex5` by file name, so a source edit without a compile runs the previous binary.
2. **Clean.** Tester `.tst` cache and the cached `.set` are removed.
3. **Backtest.** `terminal64.exe` starts with `/config:` and writes the report.
4. **Extract.** One parse of the HTML or SpreadsheetML report. Deals go into SQLite. `.set` files must be UTF-16LE with a BOM.
5. **Analyze.** Deal statistics are written to `analysis.json`: monthly P/L, drawdown events, top losses, loss sequences, position pairs, direction bias, streaks, and the peak number of open positions. A `Layer #N` comment is stored as grid depth on the worst losses.

Three tools return as soon as the terminal is launched. Poll them with `get_backtest_status`. The background task extracts deals and does not write `analysis.json`.

| Tool | Compile | Analyze |
| --- | --- | --- |
| `run_backtest` | yes, unless `skip_compile` | yes, unless `skip_analyze` |
| `launch_backtest` | yes, unless `skip_compile` | no |
| `run_backtest_quick` | no | no |
| `run_backtest_only` | no | no |

After a launched run, `analyze_report` builds `analysis.json` from the stored deals.

## Tools

Registered names and arguments are in `src/tools/definitions/`. [docs/MCP_TOOLS.md](docs/MCP_TOOLS.md) is a longer narrative of the same tools and can lag those schemas. Counts match `get_tools_list()`.

| Area | Tools | Covers |
| --- | ---: | --- |
| Backtest | 9 | Full pipeline, launch modes, rolling weeks, tester journal, cache |
| Optimization | 4 | Detached genetic run, live status, parsed passes, job list |
| Analysis | 20 | Monthly P/L, drawdown, streaks, deal search, baseline comparison |
| Reports | 19 | Search, archive, history notes, best-of ranking, deal CSV export |
| Experts | 9 | Compile, list, and search EAs, indicators, and scripts |
| Set files | 8 | Read and write UTF-16LE `.set` files, diffs, sweeps, optimization export |
| System and diagnostics | 23 | Setup check, account, updates, Wine, logs, stuck processes, crash reports |

## Project layout

```text
src/
  main.rs                  CLI: stdio, --port, or --test-launch
  lib.rs                   Module index
  protocol.rs              JSON-RPC request and response types
  mcp_server.rs            Session, initialize, and tools/call
  analytics/               Report extraction and deal analysis
  compile/                 MetaEditor wrapper
  models/                  Config, deals, metrics, report records
  optimization/            Genetic runner and results parser
  pipeline/                Backtest orchestration
  storage/                 SQLite reports and deals
  utils/                   UTF-16LE reader for .set and .ini files
  tools/definitions/       Tool JSON schemas, one module per handler
  tools/handlers/          Tool implementations
tests/
  integration_tests.rs     Fixture checks run by cargo test
  fixtures/                Sample HTML, SpreadsheetML, and deals CSV
  manual/                  Sessions that need a local terminal
scripts/                   setup.sh, platform detection, build, release
config/                    Example YAML and an example .set file
docs/                      Install, config, architecture, remote agents
fuzz/                      Report-metric parser fuzz target
```

Definition modules and handler modules use the same names: `analytics`, `backtest`, `experts`, `optimization`, `reports`, `setfiles`, `system`, and `utility`. `compare_baseline` is declared in `definitions/analytics.rs` and implemented in `handlers/analytics.rs`.

## Build and test

```bash
cargo build --release
cargo test
cargo fmt --all -- --check
cargo clippy --all-targets -- -D warnings
```

`tests/manual/` talks to a built binary and a Wine prefix. `cargo test` runs `tests/integration_tests.rs` against `tests/fixtures/` and does not start MetaTrader 5.

Parser fuzzing, with `cargo-fuzz` installed:

```bash
cargo fuzz run parse_metrics
```

## Documentation

| Document | Contents |
| --- | --- |
| [docs/QUICKSTART.md](docs/QUICKSTART.md) | Install, config, and the first backtest |
| [docs/MCP_TOOLS.md](docs/MCP_TOOLS.md) | Narrative for each tool. Schemas in code win on conflict |
| [docs/CONFIG.md](docs/CONFIG.md) | Config lookup and flat keys |
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | Pipeline stages, cache rules, optimization isolation |
| [docs/REMOTE_AGENTS.md](docs/REMOTE_AGENTS.md) | Extra `metatester64.exe` workers on Linux |
| [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md) | Wine, display, and tester failures |
| [docs/INDEX.md](docs/INDEX.md) | Reading order |

## License

[MIT](LICENSE). Copyright holders accept no liability for trading losses, incorrect analysis, data loss, Wine incompatibilities, or MetaTrader 5 failures.

MetaTrader is a trademark of MetaQuotes Software Corp. You are responsible for the licenses of Wine, CrossOver, and MetaTrader 5, and for the trading rules of your jurisdiction.

## Contact

Telegram: [t.me/teraus111](https://t.me/teraus111)
