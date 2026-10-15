# Configuration

The server reads one flat text file. Keys are `name: value` on a single line. Nested YAML is ignored, because the parser splits each line on the first colon and does not build a tree.

```yaml
wine_executable: "/path/to/wine64"
terminal_dir: "/path/to/MetaTrader 5"
```

`scripts/setup.sh` writes `config/mt5-quant.yaml` in the repository. That file is gitignored. The annotated copy you can commit is [config/mt5-quant.example.yaml](../config/mt5-quant.example.yaml).

## Which file is read

`Config::writable_config_path` in `src/models/config.rs` picks the first match:

1. `$MT5_MCP_HOME/config/mt5-quant.yaml`, when `MT5_MCP_HOME` is set.
2. `config/mt5-quant.yaml` beside the executable, when that file exists.
3. `config/mt5-quant.yaml` in this crate, when that file exists. A binary built from this tree, including `cargo run`, uses the file `setup.sh` writes.
4. `~/.config/mt5-quant/config/mt5-quant.yaml`, when that file exists.
5. Beside the executable, for a portable copy that is not a Cargo `target/` build and is not installed in `/bin`, `/usr/bin`, or `/usr/local/bin`.
6. This crate's `config/` directory, when `Cargo.toml` is still on disk.
7. `~/.config/mt5-quant/config/mt5-quant.yaml`.

If the chosen file is missing, the server discovers Wine and the terminal, writes defaults, and saves that file.

Reports and `reports.db` go to `Config::installation_dir`: `$MT5_MCP_HOME` when set, otherwise the binary directory when a config file already sits beside a non-system binary, otherwise `~/.config/mt5-quant`. An absolute `reports_dir` overrides the report directory. A relative `reports_dir` does not.

## Keys

| Key | Required | Role |
| --- | --- | --- |
| `wine_executable` | yes, for compile and test | Path to `wine` or `wine64` |
| `terminal_dir` | yes | MetaTrader 5 installation directory |
| `experts_dir` | no | `MQL5/Experts`. Derived from `terminal_dir` when omitted |
| `indicators_dir` | no | `MQL5/Indicators` |
| `scripts_dir` | no | `MQL5/Scripts` |
| `tester_profiles_dir` | no | `MQL5/Profiles/Tester` |
| `tester_cache_dir` | no | `Tester` cache directory |
| `display_mode` | no | `auto`, `gui`, or `headless`. Default `auto` |
| `project_dir` | no | EA project root used by project tools |
| `backtest_symbol` | no | Symbol used when a tool call omits `symbol` |
| `backtest_deposit` | no | Default `10000` |
| `backtest_currency` | no | Default `USD` |
| `backtest_leverage` | no | Default `500` |
| `backtest_model` | no | `0` every tick, `1` 1-minute OHLC, `2` open prices. Default `0` |
| `backtest_timeframe` | no | Default `M5` |
| `backtest_timeout` | no | Seconds. Default `900` |
| `backtest_login` | no | Account login passed to the tester |
| `backtest_server` | no | Account server |
| `backtest_password` | no | Account password. Keep this file private |
| `opt_log_dir` | no | Optimization log directory. Default `/tmp` |
| `opt_min_agents` | no | Default `1` |
| `opt_max_agents` | no | `0` in code when unset. Auto-discover writes `20` |
| `reports_dir` | no | Absolute directory for report output |

`backtest_symbol` in the example file is `EURUSD`. Auto-discover writes `XAUUSD` when it creates a new file.

Comments start with `#`. Quoted and bare values are both accepted. Empty values, `null`, and `~` are skipped.
