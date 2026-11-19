# 5-Stage RISC-V RV32I Core

A synthesizable 32-bit **RISC-V RV32I** processor core written in SystemVerilog, built from scratch and timing-closed at **142 MHz** on a Xilinx Artix-7 (speed grade -1). It comes with a complete SoC wrapper (BRAM, UART, GPIO, optional VGA framebuffer), a self-checking simulation environment and a one-command Vivado build flow.

The main goal of the project is **frequency**: every stage boundary and every hazard path was shaped around the Artix-7 fabric to keep the critical path short, while staying at roughly 1.1k LUTs.

| | |
| :--- | :--- |
| **ISA** | RV32I, little-endian |
| **Pipeline** | 5-stage in-order (IF → ID → EX → MEM → WB) |
| **Fmax (post-route)** | **142 MHz** (7.0 ns), WNS +0.003 ns, WHS +0.082 ns, 0 failing endpoints |
| **Area** | 1,124 LUTs (5.4%) · 430 FFs (1.0%) · 365 slices (4.5%) · 0 DSP |
| **Memory** | 8 × RAMB36E1 (16 KB data memory + instruction memory), register file in 44 LUTRAM |
| **Target boards** | Digilent Basys 3 (`xc7a35tcpg236-1`) |
| **Tools** | AMD/Xilinx Vivado 2019.2 – 2025.2 (timing results from 2025.2) |
| **Verification** | Self-checking testbench, 50+ assembly tests, cycle-accurate CSV trace, built-in disassembler |

> Utilization is for the full SoC on the Basys 3 (`xc7a35t`), as reported by Vivado after place & route.

---

## Table of contents

1. [Microarchitecture](#microarchitecture)
2. [How 142 MHz was reached](#how-142-mhz-was-reached)
3. [Hazard handling](#hazard-handling)
4. [SoC and memory map](#soc-and-memory-map)
5. [Implementation results](#implementation-results)
6. [Verification](#verification)
7. [Quick start](#quick-start)
8. [Repository layout](#repository-layout)
9. [Limitations](#limitations)

---

## Microarchitecture

```
   ┌───────────────────────────────────────────────────────┐
   │                      HAZARD UNIT                      │
   │  rsi_comparator ─► fwd_unit ─► hazard_detection_unit  │
   └────┬──────────────┬──────────────┬──────────────┬─────┘
        │ stall_pc     │ stall_if_id  │ flush_id_ex  │ flush_ex_mem
        ▼              ▼              ▼              ▼
   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │    IF    │──►│    ID    │──►│    EX    │──►│   MEM    │──►│    WB    │
   │  PC gen  │   │ Decoder  │   │   ALU    │   │Store pack│   │Load align│
   │BRAM IMEM │   │ Imm gen  │   │ Shifter  │   │BRAM DMEM │   │ Sign-ext │
   │          │   │ RegFile  │   │ Br. unit │   │ MMIO mux │   │ RF write │
   └──────────┘   └──────────┘   └──────────┘   └──────────┘   └──────────┘
```

Hazard control is applied only at the front of the pipeline: `stall_pc` freezes the PC (IF) and `stall_if_id` holds the IF/ID register (ID). The back stages (EX, MEM, WB) are never stalled; instead `flush_id_ex` and `flush_ex_mem` turn the instruction entering EX or MEM into a bubble. Forwarding paths are omitted from the diagram; see [Hazard handling](#hazard-handling).

| Stage | What happens | Source |
| :--- | :--- | :--- |
| **IF** | PC generation; the BRAM address is driven by `pc_next`, so instruction and PC arrive in ID together with no bubble | `stages/fetch.sv`, `modules/pc.sv` |
| **ID** | 9-bit-key decoder, immediate generator (I/S/B/U/J), 32×32 register file with async read, WB→ID bypass, early load detection | `stages/decode.sv`, `modules/id.sv`, `modules/imm_gen.sv`, `memory/register_file.sv` |
| **EX** | ALU, separate barrel shifter, branch comparator/unit, 2:1 forwarding muxes | `stages/execute.sv`, `modules/alu.sv`, `modules/shifter_alu.sv`, `modules/br_unit.sv` |
| **MEM** | Store byte-enable generation (SB/SH/SW), data BRAM access, MMIO decode | `stages/memory.sv`, `memory/dmem_wr_port_m.sv` |
| **WB** | Load alignment and sign/zero extension (LB/LH/LW/LBU/LHU), register write-back mux | `stages/writeback.sv`, `memory/dmem_rd_port_m.sv` |

Implemented spec details: `x0` hardwired to zero, JALR clears bit 0 of the target, illegal opcodes raise an `illegal` flag, `FENCE`/`ECALL`/`EBREAK` execute as NOPs.

---

## How 142 MHz was reached

Closing timing at 142 MHz on a -1 speed-grade Artix-7 is a matter of removing long combinational chains. These are the specific decisions:

**1. Forwarding selects are precomputed one stage early.**
In a textbook pipeline the register-index comparators sit directly in front of the ALU operand muxes. Here, the comparisons against `rd_E/M/W` run in **Decode**, in parallel with the register-file read, and the resulting select signals are registered. In Execute they are plain flip-flop outputs, so the ALU input sees only a 2:1 mux.

**2. Load data is not forwarded into the ALU.**
The BRAM → sign-extend → mux → forward → ALU path is the classic timing killer. Only the ALU result (`alu_out_W`) is forwarded from WB to EX; load-use cases are handled by the hazard unit with a stall. Full load forwarding exists behind the `USE_DMEM_WB_EX_FORWARDING` define, which lowers CPI but breaks timing at this frequency.

**3. Load alignment lives in Writeback.**
Sign/zero extension and byte/halfword slicing are done in WB, so BRAM read latency never chains into extension logic inside MEM.

**4. Branch resolution is registered.**
The branch outcome (`jfexe`) and target (`jfpc`) are computed in EX but **registered into the EX/MEM stage**, and the PC redirect happens from MEM. This breaks the ALU/comparator → PC mux → fetch loop at the cost of a 2-cycle taken-branch penalty.

**5. Synchronous BRAM fetch without a bubble.**
`imem_addr = pc_next` and `pc_D` latches `pc_next` on the same edge the BRAM output appears, so instruction and PC stay aligned without extra registers or stall cycles.

---

## Hazard handling

| Hazard | Action | Penalty |
| :--- | :--- | :--- |
| Taken branch / JAL / JALR | Flush ID and EX, redirect PC from MEM | 2 cycles |
| Load-use (load in EX) | Stall PC and IF/ID, insert bubble | 1 cycle |
| Load-use (load in MEM) | Stall PC and IF/ID, insert bubble | +1 cycle (2 total for back-to-back use) |
| RAW, ALU result (EX → EX) | Forward `alu_out_M` | 0 |
| RAW, ALU result (MEM → EX) | Forward `alu_out_W` | 0 |
| RAW (WB → ID) | Bypass `wb_wd` in Decode | 0 |

---

## SoC and memory map

The core sits in `cpu_system.sv` together with a PLL (100 MHz board clock → 142 MHz core clock), a power-on reset generator, BRAMs and peripherals.

| Address | Target | Access | Notes |
| :--- | :--- | :--- | :--- |
| `0x0000_0000` | **DMEM** | RW | 16 KB true dual-port BRAM, per-byte write enables, byte addressable |
| `0x0000_4000` | **LEDs** | W | Word write drives `led[15:0]` |
| `0x1000_0000` | **VRAM** | W (CPU) | Optional VGA framebuffer (320×240 / 640×480), video build only |
| `0x2000_0000` | **UART** | RW | Registers listed below. In the video configuration this region also hosts the instruction memory / boot vector |
| `0x3000_0000` | **Buttons** | R | `{27'b0, BTNC, BTND, BTNL, BTNR, BTNU}` |

**UART** (115,200 baud, 8-N-1, full duplex, mid-bit RX sampling):

| Offset | Register | Description |
| :--- | :--- | :--- |
| `0x0` | `TXSTATUS` | bit 0 = transmitter ready |
| `0x4` | `TXDATA` | write a byte to transmit |
| `0x8` | `RXSTATUS` | bit 0 = received byte valid |
| `0xC` | `RXDATA` | read received byte; the read clears the valid flag |

The build script recomputes `RV_TIME_BASE` in `risc_v.svh` so the baud divider stays correct if the clock changes.

---

## Implementation results

Vivado 2025.2, Basys 3 (`xc7a35tcpg236-1`), post-route with physical optimization.

**Timing** (`cpu_system_timing_summary_postroute_physopted.rpt`)

| WNS | TNS | WHS | THS | WPWS | Failing endpoints |
| :--- | :--- | :--- | :--- | :--- | :--- |
| +0.003 ns | 0.000 | +0.082 ns | 0.000 | +2.321 ns | 0 / 1,691 |

All user-specified timing constraints are met. Setup slack is intentionally tight: the design was pushed until it just closed.

**Utilization** (`cpu_system_utilization_placed.rpt`)

| Resource | Used | Available | % |
| :--- | :--- | :--- | :--- |
| Slice LUTs | 1,124 (1,080 logic + 44 LUTRAM) | 20,800 | 5.40 |
| Flip-flops | 430 | 41,600 | 1.03 |
| Slices | 365 | 8,150 | 4.48 |
| RAMB36E1 | 8 | 50 | 16.00 |
| DSP48E1 | 0 | 90 | 0.00 |
| PLL | 1 | 5 | 20.00 |

More than 94% of the logic fabric stays free for accelerators or additional peripherals.

---

## Verification

```
  prg/uBench/*.S ──► RARS_convert_to_hex.py ──► .hex + ub.lst
                                                    │
                    rv_nsu_tb.sv + TraceLogger ◄────┘
                      • backdoor-loads IMEM per test
                      • resets CPU, clears RF and DMEM between tests
                      • logs every retired instruction to CSV
                      • PASS/FAIL read from register x31
```

- **Self-checking tests.** Each test reports through `x31`: `0` running, `1` pass, `2` fail. The testbench runs the whole list from `ub.lst` without recompiling.
- **50+ assembly micro-tests** (`prg/uBench/`) covering ALU and immediates, logic ops, all shifts (including boundary values 0–31), `LUI`/`AUIPC`, all six branch conditions, `JAL`/`JALR`, loads and stores of every width with sign extension and alignment, plus an LED hardware test.
- **Cycle-accurate trace.** For every retired instruction the logger writes time, instruction count, PC, hex encoding, disassembly and all 32 registers to `prg/uBench/hex/res/<test>.csv`.
- **Built-in disassembler** (`risc_v_sim_pkg::disasm`) turns raw instructions into GNU-style assembly in logs and waveforms.
- **SHA3-256 reference model** in pure SystemVerilog (`src/sim/sha3_256.svh`, testbench `sha3_tb.sv`) for golden-signature generation and co-simulation.

---

## Quick start

**Requirements:** Vivado (2019.2 or newer), Python 3.8+, Java 8+ (for `rars.jar`), Linux or Windows.

**1. Build the test programs**

```bash
cd prg
python3 RARS_convert_to_hex.py      # assembles uBench/*.S -> .hex and refreshes ub.lst
```

**2. Run the simulation**

Open `cfg/rv-nsu.xpr` in Vivado and run *Behavioral Simulation* with `rv_nsu_tb` as the top module. The console prints PASS/FAIL per test and traces land in `prg/uBench/hex/res/`.

**3. Build the bitstream**

```bash
chmod +x xbld.sh
./xbld.sh                           # Windows: xbld.bat
# or directly:
vivado -mode batch -journal log/bld.jou -log log/bld.log -source xbld.tcl -notrace
```

The script generates `mem_init_path.svh` and `video_config.svh`, configures the 142 MHz PLL, updates `RV_TIME_BASE`, runs synthesis, place & route, and writes timing and utilization reports to `cfg/rv-nsu.runs/impl_1/`. The default program is `prg/led_blink_kit.mem`.

---

## Repository layout

```
.
├── cfg/            Vivado project, generated headers (mem_init_path.svh, video_config.svh)
├── prg/            RARS, hex converter, default firmware, uBench/ assembly tests
├── src/
│   ├── constr/     Basys 3 and Nexys 4 DDR constraints (.xdc, .sdc)
│   ├── rtl/
│   │   ├── cpu_system.sv     SoC top: clock, reset, core, memories, peripherals
│   │   ├── cpu_core.sv       Core interconnect
│   │   ├── uart_wrapper.sv   Memory-mapped UART
│   │   ├── stages/           fetch, decode, execute, memory, writeback
│   │   ├── hazard_unit/      hazard detection, forwarding, comparators
│   │   ├── modules/          alu, shifter, branch unit, decoder, imm gen, pc
│   │   ├── memory/           register file, imem, load/store ports
│   │   ├── lib/              dual-port BRAM, reset generator, UART engine
│   │   └── include/          risc_v.svh, risc_v_sim.svh
│   └── sim/        Testbench, TraceLogger, SHA3 model
├── files.f         Compilation manifest
├── xbld.sh/.bat/.tcl   Build automation
└── test.tcl        Timing path analysis helper
```

---

## Limitations

- Base **RV32I only**: no M/A/C extensions, no CSRs, no privileged modes, no interrupts or exceptions. `ECALL`, `EBREAK` and `FENCE` are treated as NOPs.
- In-order, single-issue, no branch prediction: every taken branch costs 2 cycles.
- Load-use hazards cost 1–2 stall cycles by design (the price of the 142 MHz path).
- Verification is directed testing with self-checking assembly programs; no formal verification and no riscv-arch-test run yet.
- Timing slack is small (+0.003 ns); different Vivado versions or seeds may need re-tuning.

---

## License
- nope
