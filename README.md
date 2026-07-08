# PulseBook
![Stars](https://img.shields.io/github/stars/Shivfun99/Pulse-Order?style=for-the-badge)
![Forks](https://img.shields.io/github/forks/Shivfun99/Pulse-Order?style=for-the-badge)
![Clones](https://img.shields.io/badge/dynamic/json?color=success&label=Clones&query=count&url=https://gist.githubusercontent.com/Shivfun99/c4935a2073e2566ec90fd5c44f1a48ed/raw/clone.json&logo=github)

**PulseBook** is a C++20 + DPDK low-latency trading packet processor.

It takes compact Ethernet-style market-data packets, updates an in-memory L2 order book, runs a simple imbalance strategy, applies risk checks, and emits outbound order packets through DPDK-style RX/TX paths.

This project is mainly about learning and proving the **trading engine hot path**:

```text
market-data packet
    -> decode
    -> L2 book update
    -> strategy decision
    -> risk check
    -> order encode
    -> TX enqueue
```

It is **not** claiming exchange latency or physical NIC wire latency.

---

## Why I built this

I wanted to understand what actually matters in a low-latency trading path.

Instead of building a big backend system with APIs, databases, queues, dashboards, etc., I kept the hot path small:

- fixed-size binary packets
- integer price ticks
- fixed-memory order book
- preallocated order handling
- no logging/allocation inside the measured path
- DPDK RX/TX style processing

The goal of V1 is to validate the packet-processing engine before moving to real NIC hardware and real market-data replay.

---

## Current status

| Area | Status |
|---|---|
| C++20 trading engine | Done |
| Fixed binary market/order protocol | Done |
| 62-byte Ethernet frame encode/decode | Done |
| L2 order book | Done |
| BUY / SELL / NO_SIGNAL logic | Done |
| Inline risk checks | Done |
| DPDK Ring PMD virtual path | Done |
| DPDK AF_PACKET over private Linux `veth` | Done |
| Real supported NIC / VFIO benchmark | Not done yet |
| Real market-data replay under burst load | Future work |

---

## Architecture

```text
Inbound Ethernet market-data frame
        |
        v
DPDK RX / rte_mbuf
        |
        v
Fixed binary decoder
        |
        v
MarketUpdate
        |
        v
TradingEngine
   ┌────────────┬──────────────┬────────────┐
   v            v              v
L2 Order Book   Strategy       Risk Guard
        |
        v
Preallocated order path
        |
        v
Outbound order frame encoder
        |
        v
DPDK TX enqueue
```

---

## What the strategy does

This is intentionally simple for V1.

The engine keeps visible bid/ask liquidity in an L2 book.

```text
More bid pressure  -> BUY
More ask pressure  -> SELL
Balanced book      -> NO_SIGNAL
Risk limit hit     -> RISK_REJECT
Bad packet         -> INVALID_FRAME
```

This is not meant to be a profitable strategy. It is a controlled workload for testing the packet path.

---

## Performance results

All results below are from controlled synthetic workloads with fixed-size packets.

They are useful as **engine and packet-path microbenchmarks**, not as real market-burst p99 or exchange latency.

### 1. Virtual DPDK Ring PMD path

Measured boundary:

```text
after rte_eth_rx_burst() returned packet
    -> decode / book / strategy / risk / encode
    -> rte_eth_tx_burst() accepted order
```

| Core | p50 | p95 | p99 | p99.9 | Events | Failures |
|---:|---:|---:|---:|---:|---:|---:|
| CPU 0 | 110.844 ns | 248.397 ns | 552.217 ns | 706.464 ns | 1,000,000 | 0 |
| CPU 2 | 112.847 ns | 254.407 ns | 550.214 ns | 754.541 ns | 1,000,000 | 0 |

Classification:

```text
Virtual DPDK Ring PMD application-side RX-to-TX order-generation latency.
```

### 2. Kernel-backed DPDK AF_PACKET over private `veth`

Topology:

```text
Linux raw generator on pb_peer
    -> private veth pair
    -> pb_eng
    -> DPDK AF_PACKET RX
    -> PulseBook engine
    -> DPDK AF_PACKET TX
    -> generator validates returned order
```

Measured boundary:

```text
after AF_PACKET rte_eth_rx_burst() returned packet
    -> decode / book / strategy / risk / encode
    -> AF_PACKET rte_eth_tx_burst() accepted order
```

| Core | p50 | p95 | p99 | p99.9 | Events | Failures |
|---:|---:|---:|---:|---:|---:|---:|
| CPU 0 | 1.737 µs | 2.594 µs | 3.262 µs | 10.542 µs | 1,000,000 | 0 |
| CPU 2 | 1.832 µs | 2.903 µs | 7.822 µs | 22.061 µs | 1,000,000 | 0 |

Classification:

```text
Kernel-backed DPDK AF_PACKET application-side RX-to-TX-enqueue latency over Linux veth.
```

---

## Important measurement note

These numbers do **not** include:

- real physical NIC ingress/egress
- switch latency
- hardware timestamping
- exchange round-trip time
- real market-data burst queueing
- production feed handler behavior

They measure the application processing region only.

For real trading-style p99, the next step is replaying high-rate market data with bursts and measuring queue buildup/tail latency.

---

## Main design decisions

What mattered most:

- fixed-size binary frames instead of JSON/variable messages
- integer ticks instead of floating-point prices
- fixed-depth order book
- preallocated order/outbox memory
- no heap allocation in the hot path
- no logging/string formatting in the measured loop
- single-core run-to-completion design
- separate measurements for Ring PMD and AF_PACKET

What did not matter much yet:

- complex strategy logic
- multi-threading
- databases
- REST APIs
- dashboards
- Kafka/Redis/Docker
- physical NIC tuning, because V1 does not have a supported wired NIC path yet

---

## Hardware note

This laptop does not have a suitable wired PCIe Ethernet NIC for a true DPDK/VFIO physical benchmark.

Detected network hardware:

- Realtek PCIe Wi-Fi adapter
- USB RNDIS phone tethering

So V1 uses:

- DPDK Ring PMD for a clean virtual DPDK baseline
- DPDK AF_PACKET over a private Linux `veth` pair for a kernel-backed interface path

A real NIC benchmark is future work.

---

## Repository layout

```text
.
├── apps/
│   ├── pulsebook_dpdk_scenarios.cpp
│   ├── pulsebook_dpdk_latency_benchmark.cpp
│   ├── pulsebook_dpdk_regime_benchmark.cpp
│   ├── pulsebook_dpdk_afpacket_engine.cpp
│   ├── pulsebook_afpacket_generator.cpp
│   ├── pulsebook_dpdk_afpacket_latency.cpp
│   └── pulsebook_afpacket_benchmark_generator.cpp
├── include/pulsebook/
│   ├── book/
│   ├── common/
│   ├── dpdk/
│   ├── engine/
│   ├── order/
│   ├── risk/
│   ├── strategy/
│   └── wire/
├── tests/
├── scripts/level3/
├── docs/level3/
├── results/level3/
├── CMakeLists.txt
├── real_run.md
└── details.md
```

---

## Requirements

Tested on:

```text
Ubuntu 26.04
C++20
CMake
Ninja
DPDK 25.11.0
GCC 15.2.0
```

Install common build tools:

```bash
sudo apt update
sudo apt install -y build-essential cmake ninja-build pkg-config dpdk dpdk-dev libdpdk-dev
```

Check DPDK:

```bash
pkg-config --modversion libdpdk

find /usr/lib/x86_64-linux-gnu \
    \( -name 'librte_net_ring.so*' -o -name 'librte_net_af_packet.so*' \) \
    -print 2>/dev/null | sort
```

---

## Build

```bash
cmake -S . -B build-dpdk \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DBUILD_TESTING=ON \
    -DPULSEBOOK_ENABLE_DPDK=ON \
    -DPULSEBOOK_BUILD_BENCHMARKS=ON

cmake --build build-dpdk -j"$(nproc)"
```

---

## Run tests

```bash
ctest --test-dir build-dpdk --output-on-failure
```

Expected:

```text
100% tests passed
```

---

## Run Ring PMD scenario validation

```bash
./build-dpdk/pulsebook_dpdk_scenarios \
    -l 0 \
    --no-pci \
    --no-huge \
    --in-memory \
    --mbuf-pool-ops-name=ring_mp_mc \
    --file-prefix=pb_scenarios
```

This validates:

```text
NO_SIGNAL
BUY
SELL
RISK_REJECT
INVALID_FRAME
```

---

## Run Ring PMD latency benchmark

CPU 0:

```bash
./build-dpdk/pulsebook_dpdk_latency_benchmark \
    -l 0 \
    --no-pci \
    --no-huge \
    --in-memory \
    --mbuf-pool-ops-name=ring_mp_mc \
    --file-prefix=pb_ring_c0
```

CPU 2:

```bash
./build-dpdk/pulsebook_dpdk_latency_benchmark \
    -l 2 \
    --no-pci \
    --no-huge \
    --in-memory \
    --mbuf-pool-ops-name=ring_mp_mc \
    --file-prefix=pb_ring_c2
```

---

## Run AF_PACKET functional test

Create the private test link:

```bash
./scripts/level3/setup_veth_afpacket.sh
```

Terminal 1 — engine:

```bash
AF_PACKET_LIB="$(find /usr/lib/x86_64-linux-gnu \
    -name 'librte_net_af_packet.so' \
    -print 2>/dev/null | head -n 1)"

sudo ./build-dpdk/pulsebook_dpdk_afpacket_engine \
    -l 0 \
    --no-pci \
    --no-huge \
    --in-memory \
    --mbuf-pool-ops-name=ring_mp_mc \
    -d "${AF_PACKET_LIB}" \
    --vdev='eth_af_packet0,iface=pb_eng,qpairs=1,qdisc_bypass=1' \
    --file-prefix=pb_af_func
```

Terminal 2 — generator:

```bash
sudo ./build-dpdk/pulsebook_afpacket_generator pb_peer
```

Expected generator result:

```text
Market packets sent:       11
Outbound orders received:  1
Generated order side:      BUY
Status:                    OK
```

Clean up after AF_PACKET runs:

```bash
./scripts/level3/remove_veth_afpacket.sh
```

---

## Run AF_PACKET latency benchmark

Create the private test link first:

```bash
./scripts/level3/setup_veth_afpacket.sh
```

CPU 0 engine:

```bash
AF_PACKET_LIB="$(find /usr/lib/x86_64-linux-gnu \
    -name 'librte_net_af_packet.so' \
    -print 2>/dev/null | head -n 1)"

sudo ./build-dpdk/pulsebook_dpdk_afpacket_latency \
    -l 0 \
    --no-pci \
    --no-huge \
    --in-memory \
    --mbuf-pool-ops-name=ring_mp_mc \
    -d "${AF_PACKET_LIB}" \
    --vdev='eth_af_packet0,iface=pb_eng,qpairs=1,qdisc_bypass=1' \
    --file-prefix=pb_aflat_c0
```

In another terminal:

```bash
sudo ./build-dpdk/pulsebook_afpacket_benchmark_generator pb_peer
```

For CPU 2, repeat the engine command with:

```bash
-l 2
--file-prefix=pb_aflat_c2
```

Then clean up:

```bash
./scripts/level3/remove_veth_afpacket.sh
```

---

## Useful run guides

- `real_run.md` — exact step-by-step commands
- `details.md` — full project explanation, architecture and claims
- `docs/level3/` — design notes and benchmark scope
- `results/level3/` — saved successful output logs

---

## What this project is good for

PulseBook V1 is good for showing:

- low-level C++ systems programming
- DPDK RX/TX integration
- packet decoding/encoding
- latency-aware design
- benchmark honesty
- risk-aware trading-engine structure
- debugging real systems issues like PMD linking, mbuf sizing and interface setup

---

## What this project is not

This is not:

- a production trading system
- a profitable trading strategy
- an exchange simulator
- a full OMS/EMS
- a physical NIC latency benchmark
- a real market-burst p99 benchmark yet

---

## Future work

Next meaningful improvements:

- replay real/high-rate market data
- measure burst queueing and true tail latency
- add multi-symbol books
- add fills, cancels and replace handling
- compare AF_XDP vs AF_PACKET
- test on a machine with a supported wired DPDK NIC
- add hardware timestamps / external measurement for wire-to-wire latency
- isolate CPU cores and tune IRQ/governor settings for more stable runs

---

## Final honest summary

PulseBook V1 proves that a compact C++20 trading engine can process fixed binary market-data packets through DPDK-style paths and emit validated outbound order packets with very low application-side latency.

The best verified numbers so far are:

```text
Ring PMD virtual path:
    110.844 ns p50 / 552.217 ns p99
    1,000,000 measured order events
    0 failures

AF_PACKET over private veth:
    1.737 µs p50 / 3.262 µs p99
    1,000,000 measured order events
    0 failures
```

These are controlled engine/path benchmarks, not real exchange or physical NIC latency.
