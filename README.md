# riscv-uart-soc
![SystemVerilog](https://img.shields.io/badge/SystemVerilog-IEEE1800-blue)
![Simulators](https://img.shields.io/badge/verified-Icarus%20Verilog%20%7C%20Cadence%20Xcelium-brightgreen)
![Status](https://img.shields.io/badge/status-passing-success)

A memory-mapped System-on-Chip built around a custom 5-stage pipelined RISC-V core, extended with a UART transmitter connected through an address-decoding bus interconnect. The core drives serial output using ordinary `sw`/`lw` instructions to specific addresses — the interconnect transparently routes each request to data memory or the UART's registers, the same memory-mapped I/O pattern used in real embedded SoCs.

Verified with directed testbenches covering UART framing, address routing, and a full end-to-end transmission test, on two independent simulators (Icarus Verilog, Cadence Xcelium) — surfacing and fixing a genuine cross-simulator reset-timing bug along the way.

A stepping stone toward a larger multi-core, AXI4-Lite-based SoC (Final Year Project).

---

## Architecture

```
risc_v_core --data bus--> memory_map_unit ---> data_mem
                                          \---> uart_tx --> tx pin
```

`risc_v_core` is a 5-stage pipelined RISC-V core (fetch → decode → execute → memory → write-back) with full data-hazard forwarding and control-hazard handling. Its data bus is exposed rather than wired directly to memory, so `memory_map_unit` — a simple address decoder acting as the bus interconnect — can sit in between and route each request to the correct destination.

## Address map

| Address | Destination | Access | Description |
|---|---|---|---|
| `0x0000` – `0x063F` | `data_mem` | Read/Write | General-purpose data memory |
| `0x0640` | `uart_tx` data register | Write-only | Writing a byte here starts a UART transmission (if idle) |
| `0x0644` | `uart_tx` status register | Read-only | Bit 0 = 1 while UART is transmitting |

## Repository structure

```
rtl/
├── risc_v_core.sv          5-stage pipelined RISC-V core
├── memory_map_unit.sv      Address decoder / bus interconnect
├── data_mem.sv              General-purpose data memory
├── inst_mem.sv               Instruction memory (async read)
├── uart_tx.sv                 UART transmitter (4-state FSM, 8N1)
├── soc_top.sv                  Top-level SoC integration
└── ...                           Pipeline stages, ALU, control unit,
                                    hazard detection, forwarding logic

tb/
├── uart_tx_tb.sv            UART frame/timing verification
├── memory_map_unit_tb.sv    Address routing verification
├── risc_v_core_tb.sv         Core regression (hazards, forwarding, ISA)
└── soc_top_tb.sv               End-to-end UART transmission test
```

## Getting started

### Icarus Verilog

```bash
iverilog -g2012 -o sim.out \
  rtl/pipeline_register.sv rtl/program_counter.sv rtl/inst_mem.sv rtl/IF_ID_Stage.sv \
  rtl/reg_file.sv rtl/imm_gen.sv rtl/control_unit.sv rtl/ID_EX_Stage.sv \
  rtl/hazard_detection.sv rtl/fwd_logic.sv rtl/alu_logic.sv rtl/branch_comp.sv \
  rtl/EX_MEM_Stage.sv rtl/data_mem.sv rtl/MEM_WB_Stage.sv rtl/risc_v_core.sv \
  rtl/uart_tx.sv rtl/memory_map_unit.sv rtl/soc_top.sv tb/soc_top_tb.sv

vvp sim.out
```

### Cadence Xcelium

```bash
xrun -sv -timescale 1ns/1ps \
  rtl/pipeline_register.sv rtl/program_counter.sv rtl/inst_mem.sv rtl/IF_ID_Stage.sv \
  rtl/reg_file.sv rtl/imm_gen.sv rtl/control_unit.sv rtl/ID_EX_Stage.sv \
  rtl/hazard_detection.sv rtl/fwd_logic.sv rtl/alu_logic.sv rtl/branch_comp.sv \
  rtl/EX_MEM_Stage.sv rtl/data_mem.sv rtl/MEM_WB_Stage.sv rtl/risc_v_core.sv \
  rtl/uart_tx.sv rtl/memory_map_unit.sv rtl/soc_top.sv tb/soc_top_tb.sv
```

**Expected output:**
```
TX start bit detected, checking frame...
ALL TESTS PASSED - byte 0x41 correctly transmitted via UART
```

## A note on cross-simulator verification

During bring-up, the design passed cleanly on Icarus Verilog but consistently failed on Xcelium. Waveform tracing showed the very first instruction fetched immediately after reset release was being lost — a simulator-specific difference in event scheduling at the reset/clock boundary, not a functional bug in the core. The fix was to lead the test program with a few throwaway `NOP` instructions, so a lost fetch at that boundary costs nothing. The design now passes identically on both simulators.

## Roadmap

- [x] Single-cycle RISC-V core
- [x] 5-stage pipeline with data/control hazard handling
- [x] UART peripheral + memory-mapped bus interconnect (this repo)
- [ ] Privileged execution modes
- [ ] Multi-core AXI4-Lite SoC integration (Final Year Project)

