# Low-Power-CDC-DSP

A low-power DSP system in Verilog: a 4-tap FIR filter feeding an asynchronous clock-domain-crossing FIFO, driving a UART transmitter — with a dedicated power controller that gates the filter's clock during idle periods. Synthesized on a Cyclone IV E FPGA with full static timing and power analysis.

---

## Overview

The system processes 8-bit input samples at a fast clock domain (100MHz), filters them through a 4-tap FIR, safely crosses them into a slow clock domain (10MHz) via an asynchronous FIFO, and transmits the results over UART at 9600 baud. A power controller monitors activity and gates the fast clock during idle periods to reduce dynamic power consumption.

**Key result:** 67% reduction in dynamic power (21.15mW → 6.94mW) when clock gating is active, with zero impact on functional correctness or timing closure.

---

## Architecture

### Signal Chain
```
data_in → [FIR Filter] → [Async FIFO / CDC] → [UART TX] → tx_serial
              ↑                                                
       [Power Controller] (gates clk_fast when idle)
```

### Modules

**`fir_4tap`** — 4-Tap FIR Filter
- Real-time weighted average of the last 4 samples (coefficients 1, 2, 3, 4)
- Pure shift-and-add implementation — no DSP blocks used, per design requirements
- Output extended to 16-bit to prevent overflow

**`power_controller`** — Clock Gating Controller
- Monitors `valid_in`; after 10 idle cycles, disables the fast clock via `altclkctrl`
- Uses Quartus's dedicated global clock IP to cut the clock at the root of the routing network (not just logic-level gating)

**`async_fifo_top`** — Asynchronous FIFO (CDC)
- Standard Gray-code pointer FIFO for safe clock-domain crossing between `clk_fast` (100MHz) and `clk_slow` (10MHz)
- Submodules:
  - `sync_2ff` — double flip-flop synchronizer for Gray-coded pointers
  - `wptr_full` — write pointer + full-flag generation (Gray code comparison with MSB inversion)
  - `rptr_empty` — read pointer + empty-flag generation
  - `fifo_mem` — 16-deep memory, forced to logic-element implementation (`ramstyle = "logic"`) instead of M9K blocks

**`uart_tx_16bit`** — UART Transmitter
- Serializes 16-bit words at 9600 baud with standard Start/Stop framing
- Pulls data from the FIFO via `rinc` when idle and FIFO is non-empty

**`system_top`** — Top-Level Integration
- Wires the full chain together
- Implements per-domain synchronous reset release (2-FF reset synchronizers for both clock domains) from a single asynchronous global reset

---

## Design Decisions

- **No DSP blocks** — FIR multiplication implemented purely via bit shifts and adders, per assignment constraints
- **Gray-code CDC** — write/read pointers converted to Gray code before crossing domains, eliminating multi-bit transition hazards
- **Logic-based FIFO memory** — forced via synthesis attribute to guarantee flip-flop implementation rather than embedded memory blocks
- **Root-level clock gating** — `altclkctrl` IP cuts the clock at the global routing network root, not just at the module's enable logic, for genuine power savings
- **Synchronous reset per domain** — a single async reset is synchronized independently into each clock domain to avoid reset-related metastability

---

## Verification

**Testbench scenario:** Burst → Wait → Burst → UART transmission, verified across all four IP blocks (FIR, CDC FIFO, UART, system-level):

1. **First data burst** — 3 samples (`0xAAAA`, `0x5555`, `0x1234`) fed in, verified pipeline fill and correct `valid_out` timing
2. **Forced clock gating** — extended idle period confirmed the fast clock stops toggling while FIFO pointers hold state
3. **Wake-up / second burst** — 2 more samples fed in, verified clean recovery and continued correct operation
4. **UART transmission** — serial output verified bit-by-bit against expected framing (Start/Data/Stop)

All scenarios verified via ModelSim waveform analysis at both the individual IP level and full system level.

---

## Synthesis & Analysis Results

**Target device:** Cyclone IV E (EP4CE115F29C7), Quartus Prime 21.1.0

| Metric | Result |
|---|---|
| Logic elements | 433 / 114,480 (<1%) |
| Registers | 320 |
| Embedded multipliers | 0 / 532 (0%) — confirms no DSP usage |
| Memory bits | 0 — confirms FIFO is register-based |

**Static Timing Analysis (Slow 1200mV 85°C model):**

| Clock | Setup Slack | Hold Slack |
|---|---|---|
| clk_fast (100MHz) | +3.526 ns | +0.402 ns |
| clk_slow (10MHz) | +93.624 ns | +0.404 ns |

All paths meet timing with positive slack. Cross-domain paths correctly identified as false paths by the SDC constraints.

**Power Analysis:**

| | Without Gating | With Gating | Reduction |
|---|---|---|---|
| Core Dynamic Power | 21.15 mW | 6.94 mW | **~67%** |
| Total Thermal Power | 152.69 mW | 138.43 mW | ~9.3% |

---

## Tools

- **Language:** Verilog (behavioral + dataflow)
- **Simulation:** ModelSim
- **Synthesis, STA & Power Analysis:** Intel Quartus Prime 21.1.0 (Cyclone IV E)
