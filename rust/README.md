# hft-mm-backtester

Standalone Rust backtester for the L2 LOB + trade-print dataset under `../data/MD.zip`.
Implements the Avellaneda–Stoikov (2008) market-making strategy plus a symmetric
baseline, with a streaming CSV parser, crossing-rule fill engine, fixed-tick
scheduler, and a parallel γ-sweep. See
`docs/specs/2026-05-03-hft-mm-backtester-design.md` for the design.

## Quick start

```sh
cd hft-market-making-2026/rust
unzip ../data/MD.zip -d ../data/             # one-time
cargo build --release
cargo test
./target/release/calibrate --lob ../data/lob.csv --trades ../data/trades.csv \
    > config/calibrated.toml
./target/release/backtest  --config config/example.toml
./target/release/sweep     --config config/example.toml --gammas "0.01,0.1,1.0"
python3 scripts/plot_results.py out
```

## Layout

* `src/parser/` — streaming CSV readers (LOB snapshots, trade prints, merged iterator).
* `src/market_book.rs` — latest top-25 snapshot exposing `best_bid/ask/mid/microprice`.
* `src/own_orders.rs` — own-order ledger with `place/cancel/replace/reduce_qty`.
* `src/fill_engine.rs` — crossing-rule fills (price = our limit).
* `src/strategy/` — `Strategy` trait + Avellaneda–Stoikov + symmetric baseline.
* `src/calibration.rs` — fits σ from log-returns and (A, k) by OLS regression.
* `src/metrics.rs` — PnL, position, turnover, equity curve, summary.
* `src/engine.rs` — main event loop + fixed-tick scheduler.
* `src/report.rs` — fills.csv, equity.csv, summary.json writers.
* `src/config.rs` — TOML loader with validation.
* `src/bin/` — `backtest`, `calibrate`, `sweep` CLIs.
* `tests/` — integration tests (smoke, end-to-end) + property tests
  (`fill_engine_props`, `as_props`, `parser_merge_props`).
* `benches/` — Criterion benchmarks (`parse`, `replay`).
* `scripts/` — `plot_results.py`, `coverage.sh`.

## Tests

```sh
cargo test               # all unit + integration + property tests
cargo bench --no-run     # build benches without running
cargo bench              # run all benches (writes target/criterion/)
```

The suite includes:

* Unit tests in every module.
* Integration smoke test (`tests/replay_smoke.rs`).
* Full-pipeline E2E test (`tests/end_to_end.rs`) that emits all artifacts.
* Property tests for the fill engine, AS math invariants, and parser merge.

## Coverage

Install once:

```sh
cargo install cargo-llvm-cov
rustup component add llvm-tools-preview
```

Then:

```sh
scripts/coverage.sh           # enforces COVERAGE_MIN (default 85%)
scripts/coverage.sh --html    # also writes target/llvm-cov/html/
```

## Configuration

See `config/example.toml`. Two strategies are supported via `[strategy].kind`:

* `"symmetric"` — fixed half-spread quoting around mid.
* `"avellaneda_stoikov"` — AS quoting with parameters `sigma`, `k`, `a`,
  `gamma`, `quote_qty`, `max_abs_inventory`.

The engine supports explicit AS horizon semantics:

* `horizon_mode = "finite"` uses seconds remaining until `engine.end_us`.
* `horizon_mode = "infinite"` uses a fixed `risk_horizon_ms`, which is the
  recommended default for 24/7 crypto markets.
* `horizon_mode = "rolling"` also uses a fixed `risk_horizon_ms`, but requires
  the field to be set so experiments document the intended risk window.

If `horizon_mode` is omitted, configs with `end_us` default to `"finite"` and
open-ended configs default to `"infinite"`.
