# HydraExchange

**A deterministic, high-performance electronic exchange simulator for microstructure research, execution testing, and agent-based market simulation — with live Binance WebSocket integration.**

Built in C++20. No external dependencies except OpenSSL (for the live feed).

---

## Results

| Benchmark | Throughput | p50 | p99 |
|---|---|---|---|
| LOB Insert + Cancel | 4.51 M/s | 100 ns | 4,100 ns |
| Matching Engine E2E | 4.30 M/s | 200 ns | 600 ns |
| Memory Pool Alloc | 27.86 M/s | ~0 ns | 100 ns |
| Best Bid/Ask Query | 27.03 M/s | ~0 ns | 100 ns |

**60-second simulation:** 18,060 orders · 10,084 trades · **3,640× real-time speedup**

---

## Features

### Matching Engine
- Price-time priority continuous double auction
- Order types: Limit (GTC), Market, IOC, FOK, Post-Only
- Sequence-numbered execution reports and market data
- Full partial fill support

### Limit Order Book
- `std::map<Price, PriceLevel>` with intrusive linked-list queues
- O(1) insert, O(1) cancel, O(1) best-price query
- `MemoryPool<Order, 1M>` — zero heap allocation on hot path

### Latency Model
- Log-normal network delay (mean 40µs, cv=0.15)
- M/M/1 queue delay (service rate 5M msg/sec)
- Constant engine processing (2µs)
- Cancellation race model with closed-form success probability

### Execution Models
- Queue position tracking (fill probability = `max(0, V − Q_ahead) / Q_order`)
- Almgren-Chriss square-root market impact
- Partial fill simulation with beta-distributed fill fractions

### Agents
| Agent | Strategy |
|---|---|
| `MarketMakerAgent` | Avellaneda-Stoikov inventory-skewed quoting |
| `NoiseTraderAgent` | Poisson market orders, random side/size |
| `MomentumAgent` | Fast/slow EMA crossover with cooldown |
| `LatencyArbAgent` | Stale quote detection and sniping |

### Microstructure Analytics
- **OFI** — rolling signed order flow imbalance (normalised)
- **Kyle's λ** — online exponentially-weighted OLS price impact
- **Spread** — instantaneous, mean, and time-weighted average
- **Realised Volatility** — close-to-close, Parkinson, Rogers-Satchell
- **Alpha Decay** — IC across horizons [1s, 5s, 30s, 60s, 300s]

### Live Binance Feed
- TLS WebSocket client (OpenSSL, no external libraries)
- Combined stream: `@depth@100ms` + `@trade`
- Local order book reconstruction from incremental deltas
- Real-time dashboard: BTC/USDT mid, spread, OFI, VWAP, Kyle's λ
- No API key required — public streams only

---

## Repository Structure

```
hydra-exchange/
├── engine/
│   ├── matching_engine/     matching_engine.cpp  trade_event.cpp
│   ├── orderbook/           limit_order_book.cpp  price_level.cpp
│   ├── gateway/             order_gateway.cpp  risk_checks.cpp
│   └── market_data/         feed_publisher.cpp
├── simulation/
│   ├── latency_model/       network_delay.cpp
│   ├── execution_model/     queue_position.cpp
│   └── replay/              synthetic_orderflow.cpp
├── agents/                  market_maker.cpp  (all 4 agents)
├── analytics/               orderflow_metrics.cpp
├── binance/
│   ├── ws_client.hpp        TLS WebSocket client
│   ├── binance_stream.hpp   JSON parser + LocalOrderBook
│   └── live_analytics.hpp   Real-time metrics + dashboard
├── infrastructure/
│   ├── lockfree_queue.hpp   SPSC + MPSC lock-free queues
│   ├── ring_buffer.hpp      Disruptor-style sequenced ring
│   ├── memory_pool.hpp      Slab allocator + arena
│   └── timestamp.hpp        Cross-platform nanosecond clock
├── include/hydra/
│   ├── types.hpp            Price, Quantity, OrderId, Side, etc.
│   └── order.hpp            Order struct (96 bytes, 2 cache lines)
├── benchmarks/              throughput_benchmark.cpp
├── examples/                live_binance.cpp
├── hydra_simulation.hpp     Top-level orchestrator
├── main.cpp                 CLI entry point
└── CMakeLists.txt
```

---

## Build

### Requirements
- GCC 10+ or Clang 12+ with C++20 support
- CMake 3.20+
- OpenSSL (for live feed only)

### Linux / macOS
```bash
sudo apt install cmake g++ libssl-dev   # Ubuntu
brew install cmake openssl              # macOS

git clone https://github.com/chrislernunes/hydra-exchange
cd hydra-exchange
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release
cmake --build . -j$(nproc)
```

### Windows (MSYS2 UCRT64)
```bash
pacman -S mingw-w64-ucrt-x86_64-cmake mingw-w64-ucrt-x86_64-gcc mingw-w64-ucrt-x86_64-openssl

cd "/c/Users/YourName/hydra-exchange"
mkdir build && cd build
cmake .. -DCMAKE_BUILD_TYPE=Release -G "Unix Makefiles"
cmake --build . -j4
```

---

## Usage

### Simulation modes

```bash
# Manual matching demo — watch orders cross, fill, IOC cancel
./build/hydra --example

# Execution models — queue position, market impact, cancel race
./build/hydra --exec-model

# 60-second agent simulation — 3 MMs + 15 noise + 2 momentum
./build/hydra --sim

# Throughput and latency benchmarks
./build/hydra --bench

# Run everything
./build/hydra --example --exec-model --sim --bench
```

### Live Binance feed

```bash
./build/hydra_live BTCUSDT    # Bitcoin
./build/hydra_live ETHUSDT    # Ethereum
./build/hydra_live SOLUSDT    # Solana
./build/hydra_live BNBUSDT    # BNB
```

**Sample output:**
```
--- BTCUSDT  msgs=20  trades=8 ---
  BID    84231.50  qty=0.1240  |  MID    84232.00  |  ASK    84232.50  qty=0.0890
  Spread=1.0000 USDT (0.119 bps)
  ASK: 84234.00 x 0.2210  84233.00 x 0.3150  84232.50 x 0.0890
  BID: 84231.50 x 0.1240  84231.00 x 0.1870  84230.00 x 0.2340
  VWAP=84229.31  AvgSz=0.04200 BTC  Buy=54.1%  Sell=45.9%
  OFI=+1.234 (norm=+0.214)  BookOFI=+0.163  Lambda=0.000031
  Signal: [ ^^ BUY  PRESSURE ^^ ]
```

Press `Ctrl+C` to exit.

---

## Design Notes

### Why .cpp files as headers?
The project uses a single-translation-unit include pattern — each `.cpp` file has `#pragma once` and is `#include`d directly by `main.cpp`. This avoids CMake source file management overhead for a research codebase and keeps the dependency chain explicit.

### Memory pool
`MemoryPool<Order, 1'000'000>` is heap-allocated (via `std::unique_ptr`) inside `MatchingEngine`. At 96 bytes per `Order`, the pool occupies 96 MB. Stack allocation of this size causes a segfault — the heap allocation is intentional.

### Reentrancy guard
All agent callbacks that submit orders include an `in_action_` guard. The synchronous callback chain `process → exec_report_cb → requote → process` would cause unbounded recursion without it. Fills set a `needs_requote_` flag; actual requoting is deferred to the next market data tick.

### Windows clock
`Clock::now_mono()` uses `QueryPerformanceCounter` on Windows rather than `CLOCK_MONOTONIC`, giving sub-microsecond resolution. `gmtime_r` is replaced with `gmtime_s` (reversed argument order on Windows).

---

## Key Files

| File | Purpose |
|---|---|
| `engine/orderbook/limit_order_book.cpp` | Core LOB: price map, intrusive list, match loop |
| `engine/matching_engine/matching_engine.cpp` | Order lifecycle, pool allocation, callbacks |
| `infrastructure/memory_pool.hpp` | O(1) slab allocator, no heap calls on hot path |
| `infrastructure/lockfree_queue.hpp` | SPSC (Lamport) and MPSC (Dmitry Vyukov) queues |
| `simulation/replay/synthetic_orderflow.cpp` | CST stochastic LOB model event generator |
| `analytics/orderflow_metrics.cpp` | OFI, Kyle λ, spread, RV, alpha decay — all online |
| `binance/ws_client.hpp` | From-scratch TLS WebSocket (OpenSSL + Winsock/BSD) |
| `binance/binance_stream.hpp` | Dependency-free JSON parser for Binance streams |


## License

MIT License — free to use for research and commercial purposes with attribution.

---

*Feedback and pull requests welcome.*
