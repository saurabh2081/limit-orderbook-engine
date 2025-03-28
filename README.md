# High-Performance Limit Order Book (LOB) Engine — C++20

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-brightgreen.svg)](CONTRIBUTING.md)
[![CI/CD](https://github.com/saurabh2081/limit-orderbook-engine/actions/workflows/release.yml/badge.svg?branch=main)](https://github.com/saurabh2081/limit-orderbook-engine/actions/workflows/release.yml)
[![Docker Ready](https://img.shields.io/badge/Docker-Ready-blue.svg?logo=docker)](https://hub.docker.com/)
[![Benchmarks](https://img.shields.io/badge/Benchmarks-Reproducible-important.svg)](docs/bench.md)

A production-grade, ultra-low-latency C++20 Limit Order Book and matching engine engineered for quantitative trading, simulation, and backtesting. Designed from the ground up for mechanical sympathy, cache efficiency, and deterministic execution semantics.

---

### ⚡ Key Metrics
- **Throughput**: **20.7M+ messages / second** sustained hot-path execution
- **Latency**: **p50 = 0.04 µs**, **p90 = 0.08 µs**, **p99 ≈ 1.0 µs**
- **Memory Footprint**: Slab-allocated arena with $O(1)$ allocation and zero runtime heap fragmentation
- **Real-World Validation**: Reconstructed and verified against live tick data (BTCUSDT Binance US)

---

## 🧰 Tech Stack & Tooling

| Domain | Technologies & Libraries |
| :--- | :--- |
| **Core Systems** | C++20, GCC 11+, Clang 14+, CMake 3.20+, Linux (perf, valgrind, ASan) |
| **Python Ecosystem** | Python 3.11, PyBind11, NumPy, pandas, PyArrow (Parquet) |
| **Testing & CI** | Catch2, cmocka, GitHub Actions, Docker multi-stage builds |
| **Analytics & Viz** | Streamlit, Matplotlib, Scikit-learn (k-means impact clustering) |

---

## 🔎 Engineering Highlights

- **`BookCore` Engine**: Limit & market orders, instantaneous cancellations, in-place modifications, strict price-time priority (FIFO).
- **Execution Flags**: Immediate-Or-Cancel (`IOC`), Fill-Or-Kill (`FOK`), `POST_ONLY`, Self-Trade Prevention (`STP`).
- **Memory Subsystem**: Custom 1 MiB slab memory pool eliminates dynamic memory allocation on the critical path.
- **Microarchitecture Optimizations**:
  - Side-specialized matching templates to eliminate branch mispredictions in inner matching loops.
  - Intrusive doubly-linked queues per price level for $O(1)$ order enqueue/dequeue without node indirection.
  - Dedicated cache-hot best-bid/best-ask level pointers.
  - Compilation flags: `-O3 -march=native -fno-exceptions -fno-rtti`.
- **Market Data Pipeline**:
  - WebSocket & REST ingestion pipeline capturing live crypto order book depth and trade feeds.
  - Gap-aware Level-2 book reconstruction with checksum-validated state synchronization.
  - Real-time and $N\times$ accelerated historical replay engine emitting TAQ quotes and trade prints.
- **Quantitative Analytics & Backtesting**:
  - Institutional execution algos: **TWAP**, **VWAP**, **POV**, and **Iceberg**.
  - Market microstructure analytics: realized volatility (Parkinson & Garman-Klass), trade impact curves, order-flow autocorrelation, and imbalance drift.
  - Interactive Streamlit dashboard for real-time order book inspection and strategy simulation.

---

## 🧭 System Architecture

### Matching Engine Data Flow
```text
  Incoming Orders (New / Cancel / Modify / Market)
                         │
                         ▼
                ┌──────────────────┐
                │     BookCore     │
                │  Matching Engine │
                └────────┬─────────┘
                         │
          ┌──────────────┴──────────────┐
          ▼                             ▼
┌───────────────────┐         ┌───────────────────┐
│ PriceLevels (Bid) │         │ PriceLevels (Ask) │
│  best_bid cache   │         │  best_ask cache   │
└─────────┬─────────┘         └─────────┬─────────┘
          │                             │
          ▼                             ▼
┌───────────────────┐         ┌───────────────────┐
│  LevelFIFO Queue  │         │  LevelFIFO Queue  │
│ [Node]<->[Node]   │         │ [Node]<->[Node]   │
└───────────────────┘         └───────────────────┘
          │                             │
          └──────────────┬──────────────┘
                         ▼
                ┌──────────────────┐
                │ Event / Snapshot │
                │ Binary & JSONL   │
                └──────────────────┘
```

### Ladder & Queue Memory Layout
```text
Bids Ladder (descending price priority)      Asks Ladder (ascending price priority)
best_bid ──> [px = 100][FIFO] ──> ...        best_ask ──> [px = 101][FIFO] ──> ...

Intrusive LevelFIFO:
  HEAD <───> [OrderNode] <───> [OrderNode] <───> ... <───> TAIL  (O(1) insertion/removal)
```

### Custom Slab Allocator (Arena)
```text
┌───────────────────────────── 1 MiB Slab Buffer ─────────────────────────────┐
│ [OrderNode 0] │ [OrderNode 1] │ [OrderNode 2] │ ... │ [OrderNode N]          │
└─────────────────────────────────────────────────────────────────────────────┘
       ▲
       └── Free-list pointer: O(1) allocation and recycling (zero malloc/free jitter)
```

---

## 🗂️ Repository Structure

```text
├── cpp/
│   ├── include/lob/
│   │   ├── book_core.hpp        # Core matching engine declarations and fast path
│   │   ├── price_levels.hpp     # Contiguous array and sparse hash ladders
│   │   ├── types.hpp            # Typedefs (Tick, Quantity, OrderId, Flags)
│   │   ├── mempool.hpp          # Fixed-size slab memory arena
│   │   └── logging.hpp          # Snapshot serializer & event sink
│   ├── src/
│   │   ├── book_core.cpp        # Matching algorithm & side specialization
│   │   ├── price_levels.cpp     # Price ladder support routines
│   │   ├── logging.cpp          # Binary snapshot & replay log implementation
│   │   └── util.cpp             # Utility helpers
│   ├── tools/
│   │   ├── bench.cpp            # Synthetic benchmarking binary
│   │   └── replay.cpp           # Historical snapshot & feed replay tool
│   └── CMakeLists.txt           # C++ library & binary build definitions
├── python/
│   ├── olob/                    # Python analytics and execution toolkit
│   │   ├── cli.py               # CLI tool entrypoints
│   │   ├── metrics.py           # Microstructure summary calculations
│   │   ├── microstructure.py    # Volatility, impact curves, flow autocorrelation
│   │   └── sweep.py             # Parallel strategy parameter grid sweep
│   └── _bindings.cpp            # PyBind11 C++ bindings
├── docs/                        # Technical documentation, figures, and benchmark logs
├── analytics/                   # Generated microstructure plots and summaries
├── app.py                       # Interactive Streamlit order book playback dashboard
├── Dockerfile                   # Multi-stage production container build
└── CMakeLists.txt               # Top-level build configuration
```

---

## 🛠️ Build and Testing

### Prerequisites
- C++20 compliant compiler (GCC 11+ or Clang 14+)
- CMake 3.20+
- Python 3.11+ (optional, for analytics and Python bindings)

### Compiling from Source

```bash
# Clone the repository
git clone git@github.com:saurabh2081/limit-orderbook-engine.git
cd limit-orderbook-engine

# Configure Release build
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release \
      -DLOB_BUILD_TESTS=ON \
      -DLOB_PROFILING=ON

# Compile in parallel
cmake --build build -j$(nproc)
```

#### CMake Options
| Flag | Default | Description |
| :--- | :--- | :--- |
| `LOB_BUILD_TESTS` | `ON` | Builds Catch2 unit test suites |
| `LOB_PROFILING` | `ON` | Enables frame pointers (`-fno-omit-frame-pointer -g`) for profiler analysis |
| `LOB_ENABLE_ASAN` | `OFF` | Enables AddressSanitizer and LeakSanitizer in Debug mode |
| `LOB_LTO` | `OFF` | Enables Link Time Optimization (Release mode) |

### Running Unit Tests
```bash
ctest --test-dir build --output-on-failure
```

---

## ⚡ Performance Benchmarks

### Synthetic Latency & Throughput

The benchmark harness tests `BookCore` under mixed workloads of submissions, cancellations, and aggressive fills:

```bash
# Maximum throughput test (8M messages)
./build/cpp/bench_tool \
  --msgs 8000000 --warmup 200000 \
  --band 1000000:1000064:1 \
  --latency-sample 0
```

**Results:**
```text
msgs=8000000, time=0.386s, rate=20,714,662.3 msgs/sec
```

```bash
# Latency profiling with cycle-accurate sampling
./build/cpp/bench_tool \
  --msgs 8000000 --warmup 200000 \
  --band 1000000:1000064:1 \
  --latency-sample 1024
```

| Percentile | Latency (µs) |
| :--- | :--- |
| **p50** | **0.042 µs** |
| **p90** | **0.083 µs** |
| **p99** | **1.012 µs** |
| **p99.9** | **96.4 µs** |
| **p99.99** | **176.2 µs** |

Detailed percentiles and histograms are output to [`docs/bench.csv`](docs/bench.csv) and [`docs/hist.csv`](docs/hist.csv).

---

## 🔬 Profiling & Inspection

### Linux `perf`
```bash
# Run performance counter stat analysis
perf stat -d ./build/cpp/bench_tool --msgs 2000000

# Sample CPU call graphs
perf record -g -- ./build/cpp/bench_tool --msgs 2000000
perf report
```

### Valgrind Memory Safety
```bash
valgrind --tool=memcheck --leak-check=full ./build/cpp/lob_tests
```

---

## 🌐 Live Market Data Connector & Reconstruction

The repository includes a Python-driven market data pipeline to ingest, reconstruct, and validate live exchange books.

### 1. Ingestion from Live WebSocket & REST (Binance US)
```bash
lob crypto-capture \
  --exchange binanceus \
  --symbol BTCUSDT \
  --minutes 60 \
  --raw-dir raw \
  --snapshot-every-sec 60
```
This stores structured compressed JSONL feeds under `raw/YYYY-MM-DD/binanceus/BTCUSDT/`.

### 2. Normalization to Columnar Parquet
```bash
lob normalize \
  --exchange binanceus \
  --date $(date -u +%F) \
  --symbol BTCUSDT \
  --raw-dir raw \
  --out-dir parquet
```

### 3. Gap-Aware Book Reconstruction
```bash
python -m orderbook_tools.reconstruct \
  --raw-dir raw --date $(date -u +%F) --exchange binanceus --symbol BTCUSDT \
  --out-dir recon --tick-size 0.01 \
  --snap-glob "*depth/snapshot-*.json.gz" \
  --diff-glob "*depth/diffs-*.jsonl.gz"
```
Reconstructs the full Level-2 ladder by sequentially applying incremental depth deltas onto REST book snapshots, resyncing automatically on packet drops or sequence discontinuities.

---

## ▶️ Accelerated Historical Replay Engine

The C++ replay tool streams historical events into `BookCore` with variable clock multipliers (e.g., $1\times$, $10\times$, $50\times$) and periodic quote sampling:

```bash
# Convert normalized Parquet to CSV stream
python - <<'EOF'
import pandas as pd
df = pd.read_parquet("parquet/2025-08-25/binanceus/BTCUSDT/events.parquet")
if 'ts_ns' not in df.columns:
    df['ts_ns'] = pd.to_datetime(df['ts'], utc=True).view('int64')
df['type'] = df['type'].astype(str).str.lower()
df['side'] = df['side'].astype(str).str.lower().map({'b':'B','bid':'B','buy':'B','a':'A','ask':'A','sell':'A','s':'A'}).fillna('A')
df[['ts_ns','type','side','price','qty']].to_csv("parquet_export.csv", index=False)
EOF

# Replay at 50x speed and sample top-of-book TAQ quotes every 50ms
./build/cpp/replay_tool \
  --file parquet_export.csv \
  --speed 50x \
  --cadence-ms 50 \
  --quotes-out taq_quotes.csv \
  --trades-out taq_trades.csv
```

Outputs generated:
- `taq_quotes.csv`: Regular time grid sampling of `[ts_ns, bid_px, bid_qty, ask_px, ask_qty]`
- `taq_trades.csv`: Discrete trade execution prints

---

## 📊 Market Microstructure Analytics

Compute institutional-grade microstructure metrics directly from TAQ quotes and reconstructed depth:

```bash
python -m olob.microstructure \
  --quotes taq_quotes.csv \
  --trades taq_trades.csv \
  --depth-top10 recon/2025-08-25/binanceus/BTCUSDT/top10_depth.parquet \
  --plots-out analytics/plots \
  --out-json analytics/microstructure_summary.json \
  --bar-sec 60 --rv-window 30 \
  --impact-horizons-ms 500,1000 \
  --autocorr-max-lag 50 \
  --drift-grid-ms 1000
```

### Calculated Metrics
- **Realized Volatility**: High-frequency mid-price Parkinson and Garman–Klass volatility estimators.
- **Price Impact**: Future mid-price displacement (basis points) relative to trade size deciles across 500ms and 1000ms horizons.
- **Order-Flow Persistence**: Autocorrelation profile of signed trade volume across tick lags.
- **Imbalance Drift**: Future price drift conditioned on Level-1 volume imbalance deciles.
- **Impact Regimes**: $k$-means clustering across empirical price impact curve dynamics.

---

## 📈 Execution Algorithms & Backtesting

Simulate institutional parent-order execution algorithms with queue position awareness, maker/taker fee schedules, and slippage modeling.

### Supported Schedulers
1. **TWAP (Time-Weighted Average Price)**: Slices orders evenly across fixed time intervals.
2. **VWAP (Volume-Weighted Average Price)**: Dynamically weights slice sizing by historical and intraday volume profiles.
3. **POV (Percentage of Volume)**: Participates at a predetermined target percentage of market volume.
4. **Iceberg**: Dynamically replenishes visible display quantities from hidden inventory.

### Running a Backtest
```bash
lob backtest \
  --strategy docs/strategy/vwap.yaml \
  --quotes taq_quotes.csv \
  --trades taq_trades.csv \
  --out out/backtests/vwap_run \
  --seed 123
```

### Execution Strategy Benchmark Comparison

Tested over an identical 1-hour BTCUSDT order flow window (Binance US):

| Strategy | Filled Qty | Avg Price | Total Notional | Fees (USD) | Net Cost | Total PnL | Max Drawdown | Turnover | Sharpe Ratio |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **TWAP** | 5.1000 | 110,506.59 | 563,583.62 | 112.72 | 563,696.34 | -555,095.27 | 546,299.92 | 1.00 | 350.93 |
| **VWAP** | 5.1000 | 110,499.67 | 563,548.30 | 112.71 | 563,661.01 | -555,089.00 | 554,420.84 | 1.00 | 498.22 |
| **POV** | 0.9925 | 110,444.31 | 109,615.98 | 21.92 | 109,637.91 | -57,830.99 | 56,724.40 | 1.00 | 334.48 |
| **Iceberg** | 5.0999 | 110,493.91 | 563,507.91 | 112.70 | 563,620.61 | -553,977.94 | 478,685.39 | 1.00 | 202.18 |

---

## 🎥 Interactive Live Playback Dashboard

The engine includes a full **Streamlit application** to inspect the evolving book, top-of-book spread dynamics, and trade fills in real time:

```bash
streamlit run app.py
```

### Features
- Interactive playback controls: Play/Pause, $1\times$ to $100\times$ speed multipliers, variable lookback windows.
- Dual-panel charting: Midprice vs. Microprice spread evolution synchronized with child order execution fills.
- Cumulative inventory, cash balance, and mark-to-market equity curve monitoring.

---

## 📸 Snapshot Replay Equivalence Proof

To verify complete state determinism, the engine guarantees that cutting a state snapshot at timestamp $T_{\text{cut}}$ and resuming replay from that snapshot yields bit-for-bit identical executions to a continuous single-pass replay:

```bash
lob snapshot-proof \
  --events parquet/2025-08-25/binanceus/BTCUSDT/events.csv \
  --cut-ns 1724544000000000000 \
  --out out/snapshot_proof \
  --strategy docs/strategy/twap.yaml
```

The validation suite verifies:
- Identical top-of-book quote trajectories (`quotes_A.csv` == `quotes_B.csv`)
- Identical fill sequences (`fills_A.csv` == `fills_B.csv`)
- Strictly matching deterministic SHA-256 state hashes (`equivalence.json: {"ok": true}`)

---

## 🐳 Docker Deployment & GHCR Pipeline

A multi-stage, production-ready `Dockerfile` compiles the C++ matching engine and packages the Python analytics runtime in an isolated, minimal environment.

### Local Docker Build
```bash
docker build -t limit-orderbook-engine:latest .
```

### One-Command Automated Analysis Container
```bash
docker run --rm \
  -v "$PWD/parquet:/data/parquet:ro" \
  -v "$PWD/recon:/data/recon:ro" \
  -v "$PWD/out:/out" \
  limit-orderbook-engine:latest analyze \
  --exchange binanceus --symbol BTCUSDT \
  --date 2025-08-25 --hour-start 03:00 \
  --parquet-dir /data/parquet \
  --out-reports /out
```

---

## 📄 License & Attribution

This project is licensed under the [MIT License](LICENSE).

Developed and maintained by [saurabh2081](https://github.com/saurabh2081). Pull requests, suggestions, and feedback are welcome!
