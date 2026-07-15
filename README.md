# Intelligent AXI4 Memory Fabric for RISC-V SoCs

Adaptive-scheduling AXI4 interconnect IP: CPU + DMA masters → Fabric → AXI SRAM slave(s).
Synthesizable SystemVerilog, Verilator-simulatable, parameterized for FPGA/ASIC reuse.

## Status

**All 10 RTL blocks + top-level integration are written.** Verification and Python
automation scaffolding are in place but have explicit known gaps (below) before this
is sim-clean.

**RTL (complete):**
- `rtl/pkg/axi4_fabric_pkg.sv` — parameters, enums, `xact_meta_t`, `sched_score_in_t`.
- `rtl/pkg/axi4_if.sv` — AXI4 interface (AW/W/B/AR/R), master/slave/monitor modports, baseline SVA.
- `rtl/fabric/txn_manager.sv` — per-master ingest, metadata table, aging, lifecycle status.
- `rtl/fabric/txn_queue.sv` — index FIFO, per-(master,direction) instance.
- `rtl/qos/qos_engine.sv` — multi-factor scoring (QoS/wait/occupancy/outstanding/burst),
  starvation flagging with saturating boost.
- `rtl/scheduler/adaptive_scheduler.sv` — RR, fixed-priority, WRR, age-based, adaptive
  (blended score, falls back to strict oldest-first under starvation).
- `rtl/arbiter/arbiter.sv` — turns a scheduler grant into an actual AW/AR issue, holds
  the winner stable across slave stalls, reports `grant_consumed` back to the scheduler.
- `rtl/decoder/addr_decoder.sv` — parameterized base/mask address-map decode.
- `rtl/otm/otm.sv` — per-master outstanding count + backpressure, live-entry CAM for
  response-matching sanity checks.
- `rtl/response/response_mgr.sv` — B/R demux back to originating master by fabric-tagged
  ID, drives retirement pulse.
- `rtl/perf/perf_monitor.sv` — counter bank: xact counts, latency sum/max, starvation
  events, backpressure cycles, per-mode usage, max occupancy/outstanding.
- `rtl/regs/cfg_regs.sv` — memory-mapped runtime config (scheduler mode, weights,
  starvation threshold, per-candidate fixed-priority/WRR-weight arrays) + counter mirror.
- `rtl/fabric/axi4_fabric_top.sv` — wires all of the above into one fabric instance.

**Verification (scaffolded, not yet closing):**
- `tb/sva/axi4_protocol_checker.sv` — bindable AXI4 protocol checker: X/Z checks, WRAP
  burst-length legality, WLAST/AWLEN beat-count matching, bounded-time liveness checks.
- `tb/rtl_wrappers/fabric_flat_wrapper.sv` — flattens the interface-array top-level ports
  into scalar signals for Verilator (2-master/1-slave default config), with a stub
  always-ready slave model.
- `tb/cpp/tb_axi4_fabric_top.cpp` — Verilator C++ harness skeleton (reset sequencing,
  VCD dump, CSV latency log scaffold). **Stimulus driving is stubbed** — see Known Gaps.
- `scripts/run_verilator.sh` — build+run script wired to the file list above.

**Python framework (functional):**
- `python/traffic_gen.py` — random/sequential/burst/mixed/cpu_heavy/dma_heavy generators.
- `python/regression_runner.py` — sweeps {5 scheduler modes × 6 patterns}, drives
  `run_verilator.sh`, collects per-config latency CSVs.
- `python/report_generator.py` — avg/max/p99 latency, throughput, Jain's fairness index,
  bar-chart comparisons, self-contained HTML dashboard + summary CSV.

## iverilog / gtkwave quickstart (WSL)

This repo was originally written/lint-checked against **Verilator** (see
`scripts/run_verilator.sh`). It also now compiles under **Icarus Verilog (iverilog) v12**
up through `fabric_flat_wrapper.sv`, with the fixes below already applied. A hand-written
iverilog-native testbench (`tb/rtl_wrappers/tb_iverilog.sv`) drives a couple of write/read
bursts through master 0 and dumps a VCD for gtkwave.

```bash
sudo apt update && sudo apt install -y iverilog gtkwave
mkdir -p sim/build sim/waves

iverilog -g2012 -D SYNTHESIS -o sim/build/axi4_fabric.vvp \
  rtl/pkg/axi4_fabric_pkg.sv \
  rtl/pkg/axi4_if.sv \
  rtl/fabric/txn_manager.sv \
  rtl/fabric/txn_queue.sv \
  rtl/qos/qos_engine.sv \
  rtl/scheduler/adaptive_scheduler.sv \
  rtl/decoder/addr_decoder.sv \
  rtl/arbiter/arbiter.sv \
  rtl/otm/otm.sv \
  rtl/response/response_mgr.sv \
  rtl/perf/perf_monitor.sv \
  rtl/regs/cfg_regs.sv \
  rtl/fabric/axi4_fabric_top.sv \
  tb/rtl_wrappers/fabric_flat_wrapper.sv \
  tb/rtl_wrappers/tb_iverilog.sv \
  -s tb_iverilog

vvp sim/build/axi4_fabric.vvp
gtkwave sim/waves/axi4_fabric_top.vcd &
```

**`-D SYNTHESIS` is required** — it skips a block of `assert property(...)` concurrent
assertions in `axi4_if.sv` that iverilog's parser doesn't support (functional logic is
unaffected; those are checks only).

### iverilog compatibility fixes applied

1. **`rtl/fabric/txn_manager.sv`** — `ar_burst` port was missing its `input` direction
   keyword, breaking port-list parsing entirely. Fixed.
2. **`rtl/scheduler/adaptive_scheduler.sv`, `rtl/qos/qos_engine.sv`,
   `rtl/response/response_mgr.sv`, `rtl/decoder/addr_decoder.sv`** — explicit `automatic`
   keyword on procedural-block variable declarations isn't supported by iverilog
   ("Overriding the default variable lifetime is not yet supported"). Stripped the
   keyword from variable declarations (left `function automatic` on function
   declarations untouched — that form is fine).
3. **`rtl/decoder/addr_decoder.sv`** — `SLAVE_BASE`/`SLAVE_MASK` were unpacked-array
   *parameters* with a replication-literal default (`'{N{...}}`). Confirmed via minimal
   repro that iverilog cannot parse unpacked-array parameter defaults in any form
   (tried `'{N{...}}`, `'{default: ...}`, no-default, and packed multi-dim — the last one
   crashes iverilog with an internal assertion). Moved these to internal
   `always_comb`-generated arrays instead (defaults to slave 0 spanning the full address
   space; extend that block for real multi-slave configs).

### Known iverilog blocker (not fixed — needs a real decision)

**`axi4_fabric_top.sv`'s top-level ports use arrays of SystemVerilog interfaces**
(`axi4_if.slave m_axi [NUM_MASTERS]`, `axi4_if.master s_axi [NUM_SLAVES]`). Confirmed
with a minimal 8-line repro that **iverilog does not support interface arrays as module
ports in any version**, including current v12 — this is a hard tool limitation, not a
code bug. Options:
- Use Verilator instead (`scripts/run_verilator.sh` already handles this correctly), **or**
- Refactor `axi4_fabric_top.sv` + `fabric_flat_wrapper.sv` to flat per-master/per-slave
  scalar interface ports (`m_axi_0`, `m_axi_1`, `s_axi_0`, ...) instead of arrays.

## Known gaps (read before assuming this is sim-ready)

1. ~~**`id → table-index` reverse lookup is missing**~~ **CLOSED.** `txn_manager.sv` now
   resolves `free_id` (the fabric-tagged global ID reported by `response_mgr` at retire
   time) to a table index itself, via a combinational priority search over its own table
   (mirroring the live-entry CAM pattern already used in `otm.sv` — no separate CAM
   storage needed since each entry already carries its own id). It also exports
   `free_hit` and `free_latency` (the entry's `wait_cycles` at retire). `axi4_fabric_top.sv`
   was updated to fan `retire_id_c` out by id instead of hardwiring `txn_free_idx` to 0,
   and `perf_monitor`'s `retire_latency_c` now comes from the matched master's
   `free_latency` instead of being hardwired to 0. Metadata slots now free correctly and
   latency capture is real.

   While closing this, a separate **pre-existing elaboration bug** in the W-channel
   passthrough (`axi4_fabric_top.sv`) was also found and fixed: it indexed
   `m_axi[]`/`s_axi[]` interface arrays with a runtime signal (`w_owner_r`), which
   Verilator correctly rejects (interface arrays only support elaboration-constant
   indices). Fixed by flattening the needed W-channel signals into plain logic arrays
   first (`m_wvalid_flat`, etc., built with constant generate-indices) and muxing those
   instead. The full RTL file list now lints clean under Verilator (`verilator --lint-only`,
   zero errors) — confirmed as part of this change.

2. **C++ harness doesn't drive real stimulus yet.** `tb_axi4_fabric_top.cpp` has the
   reset sequence, RNG, and CSV scaffold, but the actual `dut->m0_awvalid = ...` driving
   loop is commented out pending a decision on stimulus format (inline randomization vs.
   reading `traffic_gen.py`'s CSV directly in C++). Now the biggest blocker to a passing
   regression.
3. **No scoreboard yet.** The SVA checker verifies handshake/burst legality but there's no
   golden-model scoreboard comparing expected vs. actual data/response per transaction.
4. **Slave-side model is a stub** — always-ready, same-cycle-response fake in
   `fabric_flat_wrapper.sv`. Fine for smoke-testing the grant path, not for
   burst/backpressure testing. Needs a real behavioral SRAM model with configurable latency.
5. **cfg_regs sweep isn't wired through the harness** — `regression_runner.py` sweeps
   scheduler mode in name only today; nothing in the C++ harness issues cfg writes yet.

None of these remaining gaps are architectural problems — they're the remaining "make it
actually run" wiring, and are the natural next session's work in the order listed above
(closing gap #2 next unblocks a first real passing regression).

## Folder structure

```
axi4_fabric/
├── rtl/
│   ├── pkg/         axi4_fabric_pkg.sv, axi4_if.sv
│   ├── fabric/       txn_manager.sv, txn_queue.sv, axi4_fabric_top.sv (pending)
│   ├── scheduler/    adaptive_scheduler.sv (pending)
│   ├── qos/          qos_engine.sv (pending)
│   ├── arbiter/      arbiter.sv (pending)
│   ├── decoder/      addr_decoder.sv (pending)
│   ├── otm/          otm.sv (pending)
│   ├── response/     response_mgr.sv (pending)
│   ├── perf/         perf_monitor.sv (pending)
│   └── regs/         cfg_regs.sv (pending)
├── tb/               testbenches + SVA (pending)
├── sim/              Verilator build/run scripts (pending)
├── scripts/          misc automation (pending)
├── python/           traffic-gen + regression + reporting framework (pending)
└── docs/             architecture, timing diagrams, verification plan (pending)
```

## Design conventions used throughout

- Every module `import axi4_fabric_pkg::*;` — no local re-declaration of shared types.
- Queues store table *indices*, never duplicate `xact_meta_t` — single source of truth.
- Synchronous, active-low async-reset (`rstn`) throughout; no inferred latches
  (every `always_comb` branch has a default assignment first).
- Status transitions (`xact_status_e`) are the single lifecycle model shared by
  txn_manager, scheduler, OTM, and response manager — nobody invents a parallel FSM.

## Next step

Say the word and I'll build the **QoS Engine + Adaptive Scheduler** next (items 1-2above) —
that's the core-innovation block and the one most worth getting right before the
arbiter/decoder plumbing, which is comparatively mechanical.
