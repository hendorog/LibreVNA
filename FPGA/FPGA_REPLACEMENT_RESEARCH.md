# LibreVNA FPGA Replacement Research

## Goal

Replace the Xilinx Spartan-6 FPGA (which requires the proprietary, GUI-centric Xilinx ISE 14.7
toolchain) with an FPGA that has an open-source or at least fully command-line-driven toolchain,
enabling automated development iteration with Claude Code and CI/CD integration.

---

## Current Design: Spartan-6 XC6SLX9-2-TQG144

### Part Details

| Parameter        | Value                                     |
|------------------|-------------------------------------------|
| Part Number      | XC6SLX9-2-TQG144                         |
| Family           | Xilinx Spartan-6                          |
| Logic Cells      | 9,152 (5,720 LUT6, 11,440 flip-flops)    |
| DSP Slices       | 16 available (4-6 used: DSP48A1, 18x18)  |
| Block RAM        | 32x 18Kb = 576 Kb total (~180 Kb used)   |
| Package          | 144-pin TQFP (hand-solderable)            |
| I/O Pins Used    | 68 of ~102 available                      |
| I/O Standard     | LVCMOS33 (3.3V) on all pins               |
| Input Clock      | 16 MHz external oscillator                |
| Internal Clock   | 102.4 MHz via PLL                         |
| HDL Language     | VHDL (~4,500 lines)                       |
| Build Tool       | Xilinx ISE 14.7 (end-of-life, no Linux support past RHEL 6) |

### What the FPGA Does

The Spartan-6 is the central signal processing and control hub:

- **ADC Sampling**: Drives three MCP33131 16-bit ADCs at 800 kHz (Port1, Port2, Reference)
- **DFT Engine**: 96-bin Discrete Fourier Transform with 4 parallel multiply-accumulate DSP operations, selectable window functions (Hann, Hamming, Blackman, Rectangle)
- **PLL Control**: Configures two MAX2871 frequency synthesizers (Source + LO1) for 25 MHz - 6 GHz
- **SPI Interfaces**: Slave to MCU (~42.5 MHz), masters to PLLs and ADCs
- **Hardware Control**: Digital attenuator (7-bit RFSA3714), RF switches, port routing, filter selection, LEDs, trigger I/O
- **Memory**: Sweep config BRAM (4,501x96-bit) + result BRAM (256x192-bit)

### Estimated Resource Usage

| Resource     | Available | Used (est.) | Utilization |
|--------------|-----------|-------------|-------------|
| LUTs         | 5,720     | ~2,300-3,400| 40-60%      |
| Flip-Flops   | 11,440    | ~3,400-5,700| 30-50%      |
| DSP48 Slices | 16        | 4-6         | 25-38%      |
| Block RAM    | 576 Kb    | ~180 Kb     | ~31%        |
| I/O Pins     | ~102      | 68          | 67%         |

### Minimum Requirements for Replacement

| Resource     | Minimum   | Comfortable | Notes                              |
|--------------|-----------|-------------|------------------------------------|
| LUTs (4-input equiv.) | 5,000 | 8,000+ | 2x headroom over estimated usage |
| Flip-Flops   | 6,000     | 10,000+    | Generous for control logic          |
| DSP (18x18)  | 6         | 16+        | For DFT multiply-accumulate         |
| Block RAM    | 256 Kb    | 512 Kb+    | Sweep config + result storage       |
| User I/O     | 68        | 80+        | All at 3.3V LVCMOS                  |
| PLL          | 1         | 2+         | 16 MHz -> ~100 MHz                  |

---

## Candidate FPGAs Evaluated

### 1. Lattice ECP5 LFE5U-12F / LFE5U-25F (RECOMMENDED)

**Verdict: Best overall choice. Production-grade open-source toolchain, pin-compatible upgrade path, available in TQFP-144.**

| Parameter        | LFE5U-12F-xTG144  | LFE5U-25F-xTG144  |
|------------------|--------------------|--------------------|
| LUTs (4-input)   | 12,000             | 24,000             |
| Flip-Flops       | 12,000             | 24,000             |
| DSP18x18 Slices  | 28                 | 28                 |
| Block RAM        | 460 Kb (EBR)      | 1,008 Kb (EBR)    |
| User I/O (TG144) | 98                 | 98                 |
| PLLs             | 2                  | 2                  |
| Package          | 144-TQFP           | 144-TQFP           |
| Process          | 40 nm              | 40 nm              |
| Price (DigiKey)  | ~$14               | ~$20-31            |
| In Stock         | Yes                | Yes                |

**Open-Source Toolchain (Production-Grade):**
```
VHDL/Verilog -> Yosys (+ ghdl-yosys-plugin for VHDL) -> nextpnr-ecp5 -> ecppack -> openFPGALoader
```

- **Yosys + Project Trellis + nextpnr-ecp5**: The second most mature open-source FPGA flow after iCE40. Hardware-proven in neural network accelerators and RISC-V SoCs booting Linux.
- All tools are truly open source (ISC/MIT licensed), ~500 MB install via OSS CAD Suite.
- No license servers, no vendor accounts, no GUI required.
- Runs on Linux x86_64, ARM64, macOS, and Windows.
- **VHDL Support**: The `ghdl-yosys-plugin` enables VHDL synthesis through Yosys. It includes ECP5 component libraries. Status is "experimental" but working — simple to moderately complex designs synthesize successfully. The plugin is bundled in OSS CAD Suite.
- **Vendor Fallback**: Lattice Diamond (free, CLI-capable) provides a production-grade alternative if the open-source flow has issues with specific constructs.

**Why ECP5 is the best fit:**
- Same TQFP-144 package class as the current Spartan-6 — minimal PCB redesign
- 98 user I/O in TQFP-144 (need 68) — ample headroom
- 3.3V LVCMOS supported on all banks
- 28 DSP slices (vs 4-6 used) — massive headroom for future signal processing
- 12K-24K LUTs (vs ~3,400 used) — plenty of room to grow
- PLL can synthesize 100+ MHz from 16 MHz input
- 12F and 25F are pin-compatible in TG144 — design with 12F, upgrade to 25F if needed

**Risks:**
- VHDL synthesis via ghdl-yosys-plugin may require workarounds for BRAM inference and some complex constructs. Porting to Verilog would eliminate this risk entirely.
- No equivalent to Xilinx `STARTUP` or some specialized primitives — unlikely to be needed for this design.


### 2. Gowin GW1NR-9 (LQ144P) — Budget Alternative

**Verdict: Good budget option with LQFP-144 package. Open-source toolchain is functional but less mature than ECP5.**

| Parameter        | GW1NR-LV9LQ144    |
|------------------|--------------------|
| LUTs (LUT4)      | 8,640              |
| Flip-Flops       | 6,480              |
| DSP (18x18)      | 20                 |
| Block SRAM       | 468 Kb             |
| User Flash       | 608 Kb             |
| User I/O (LQ144) | ~80+               |
| PLLs             | 2                  |
| Embedded Memory  | 64 Mbit PSRAM      |
| Package          | LQFP-144           |
| Price            | ~$5-8 (dev board ~$15) |
| In Stock         | Yes (Mouser, Sipeed) |

**Open-Source Toolchain (Maturing):**
```
Verilog -> Yosys (synth_gowin) -> nextpnr-himbaechel -> gowin_pack -> openFPGALoader
```

- Project Apicula provides the open-source flow. Funded by NLnet/NGI0.
- GW1N-9 is one of the better-supported Gowin devices in Apicula.
- VHDL support same as ECP5 (via ghdl-yosys-plugin).
- **Vendor Fallback**: Gowin EDA with `gw_sh` CLI tool (free license, works on Linux with workarounds).

**Pros:**
- Cheapest option. Tang Nano 9K dev board is ~$15 for prototyping.
- LQFP-144 package — hand-solderable, similar to current design.
- Integrated PSRAM could be useful for extended sweep storage.
- Resources (8,640 LUTs, 20 DSP) comfortably meet requirements.

**Cons:**
- Apicula is less mature than ECP5 open-source support — may hit edge cases.
- Gowin vendor tools have inconsistent documentation and Linux quirks.
- Chinese vendor — some supply chain considerations for long-term projects.
- Fewer community resources and examples compared to Lattice or Xilinx.


### 3. Cologne Chip GateMate A1 (CCGM1A1) — Open-Source-First Option

**Verdict: Interesting newcomer where the vendor officially endorses the open-source toolchain. BGA-only is a drawback for this project.**

| Parameter        | CCGM1A1            |
|------------------|--------------------|
| Logic Elements   | 20,480 (8-input CPE) |
| Block RAM        | 1,310 Kb           |
| DSP Equiv.       | CPE-based (no hard multipliers) |
| User I/O         | 162 (BGA-324)      |
| PLLs             | 4                  |
| SerDes           | Up to 5 Gbps       |
| Package          | BGA-324 (0.8mm pitch) |
| Process          | 28 nm              |
| Price (DigiKey)  | ~$23               |
| In Stock         | Yes                |

**Open-Source Toolchain (Vendor-Endorsed):**
```
Verilog -> Yosys -> nextpnr-himbaechel (gatemate) -> Project Peppercorn -> openFPGALoader
```

- Cologne Chip now labels their proprietary PnR as "legacy" and promotes OSS CAD Suite as the primary toolchain.
- European company (Germany), 28nm process.
- Presented at CERN FPGA Developers Forum (Sept 2025).

**Pros:**
- The ONLY FPGA where the vendor actively recommends the open-source toolchain.
- 20K 8-input logic elements — very generous for this design.
- Pin-compatible upgrade path to GateMate A2 and A4.
- European supply chain.

**Cons:**
- **BGA-324 only** — requires more complex PCB (4+ layers, BGA soldering). This is the biggest drawback for a project currently using hand-solderable TQFP.
- No hard DSP multipliers — multiply-accumulate must be inferred from logic elements. May require more resources for the DFT engine.
- Smaller community than ECP5 or Gowin — fewer examples, less Stack Overflow/forum help.
- 8-input CPE architecture is unusual — harder to estimate resource usage from a 4-LUT design.


### 4. AMD Spartan-7 (XC7S15 / XC7S25) — Best Vendor-Tool Option

**Verdict: Easiest code migration (same vendor, VHDL native) but BGA-only and requires proprietary Vivado. Good if open-source toolchain is not a hard requirement.**

| Parameter        | XC7S15             | XC7S25             |
|------------------|--------------------|--------------------|
| Logic Cells      | 12,800             | 23,360             |
| LUTs (LUT6)      | 8,000              | 14,600             |
| Flip-Flops       | 16,000             | 29,200             |
| DSP48E1 Slices   | 20                 | 80                 |
| Block RAM        | 360 Kb             | 1,620 Kb           |
| User I/O         | 100                | 150                |
| PLLs/MMCMs       | 2                  | 3                  |
| Package          | CPGA196, CSGA225   | CSGA225, CSGA324   |
| Price            | ~$15-25            | ~$20-35            |
| In Stock         | Yes                | Yes                |

**Toolchain:**
```
VHDL/Verilog -> Vivado (TCL batch mode) -> bitstream
vivado -mode batch -source build.tcl
```

- Vivado ML Standard is free for all Spartan-7 and Artix-7 devices.
- Fully scriptable via TCL — no GUI required. Every operation has a CLI equivalent.
- VHDL natively supported — existing code would need minimal changes (mostly constraint file format UCF -> XDC, and any Spartan-6 primitive replacements).
- Supported through 2040.

**Pros:**
- Easiest migration path from Spartan-6. Same vendor architecture, same HDL.
- Vivado TCL scripting is extremely mature and well-documented.
- Best timing closure and optimization of any option.
- Native VHDL support — no experimental synthesis plugins.

**Cons:**
- **No TQFP-144 package** — Spartan-7 is BGA-only (smallest is CPGA196 at 8x8mm, 0.5mm pitch). Requires PCB redesign with BGA routing.
- **Not open source** — Vivado is ~80-100 GB proprietary install. Free as in beer, not freedom.
- Requires AMD account and license file (node-locked).
- Cannot easily run in CI/CD containers without license management.
- The open-source alternative (openXC7) is experimental and not production-ready.


### 5. Lattice iCE40 HX8K — NOT RECOMMENDED

**Verdict: Most mature open-source toolchain but too resource-constrained for this design.**

| Parameter        | iCE40 HX8K         |
|------------------|---------------------|
| Logic Cells      | 7,680               |
| DSP              | None (soft only)    |
| Block RAM        | 128 Kb              |
| User I/O         | 206 max             |
| PLLs             | 2                   |

- No hardware multipliers — DFT engine would consume excessive logic.
- Only 128 Kb block RAM — insufficient for sweep config + result storage.
- Would require significant design optimization to fit, with no headroom.


---

## Recommendation Summary

### Tier 1: Recommended

| Rank | FPGA | Why |
|------|------|-----|
| **#1** | **Lattice ECP5 LFE5U-25F-6TG144C** | Best balance: production-grade OSS toolchain, TQFP-144 package, ample resources (24K LUT, 28 DSP, 1008 Kb BRAM), ~$20, pin-compatible with 12F for cost optimization |
| **#2** | **Lattice ECP5 LFE5U-12F-6TG144C** | Same as above but smaller/cheaper (~$14). Still 2x-3x the resources needed. Good if cost is primary concern. |

### Tier 2: Viable Alternatives

| Rank | FPGA | Why |
|------|------|-----|
| **#3** | **Gowin GW1NR-9 (LQ144P)** | Budget option (~$5-8). LQFP-144. Adequate resources. Less mature OSS toolchain but functional. Great for prototyping with $15 Tang Nano 9K dev board. |
| **#4** | **AMD Spartan-7 XC7S25** | Best if open-source is not a hard requirement. Easiest code migration. BGA-only requires PCB redesign. Vivado CLI is excellent but proprietary. |

### Tier 3: Worth Watching

| Rank | FPGA | Why |
|------|------|-----|
| **#5** | **Cologne Chip GateMate A1** | Vendor endorses OSS toolchain (unique). BGA-only. No hard multipliers. Smaller community. Interesting for future consideration. |

---

## Migration Considerations

### HDL Language: VHDL vs Verilog

The current codebase is ~4,500 lines of VHDL. Options:

1. **Keep VHDL** — Use `ghdl-yosys-plugin` for synthesis. Works but is "experimental." Best for preserving existing investment. Works well for the ECP5 target specifically (has dedicated component library).

2. **Port to Verilog** — ~2-4 weeks of effort for this codebase size. Eliminates dependency on experimental VHDL plugin. Verilog is Yosys's native language with the best support. Recommended long-term.

3. **Gradual migration** — ghdl-yosys-plugin supports mixed VHDL/Verilog designs. New modules in Verilog, port existing modules incrementally.

### Constraint File Migration

- Current: Xilinx UCF format (`top.ucf`)
- ECP5: Lattice LPF format (similar complexity, different syntax)
- The 68 pin assignments + timing constraints would need manual translation — straightforward, ~1 hour of work.

### IP Core Migration

Current Xilinx IP cores that need replacement:

| Xilinx IP | Function | ECP5 Equivalent |
|-----------|----------|-----------------|
| PLL (Clocking Wizard) | 16 MHz -> 102.4 MHz | ECP5 EHXPLLL primitive (directly instantiable) |
| DDS Compiler (SinCos) | 12-bit phase -> 16-bit sin/cos | Replace with BRAM lookup table or CORDIC in HDL |
| DSP48 Macro | 18x18 multiply-accumulate | ECP5 MULT18X18D primitive (directly instantiable) |
| Block RAM (BRAM) | Dual-port memory | ECP5 DP16KD primitive or inferred from HDL |

### Build Automation

Example Makefile for ECP5 open-source flow:

```makefile
DEVICE = --um5g-25k
PACKAGE = TQFP144
SPEED = 6

# Using VHDL via ghdl-yosys-plugin
VHDL_SOURCES = $(wildcard src/*.vhd)

all: top.bit

top.json: $(VHDL_SOURCES)
	yosys -m ghdl -p "ghdl $(VHDL_SOURCES) -e top; synth_ecp5 -json $@"

top_pnr.config: top.json top.lpf
	nextpnr-ecp5 $(DEVICE) --package $(PACKAGE) --speed $(SPEED) \
		--json top.json --lpf top.lpf --textcfg $@

top.bit: top_pnr.config
	ecppack --compress $< --bit $@

prog: top.bit
	openFPGALoader -b <board> $<

clean:
	rm -f top.json top_pnr.config top.bit

.PHONY: all prog clean
```

This entire flow runs in ~30 seconds on modern hardware, with no GUI, no license server, and no vendor account needed.

---

## Development Board Options for Prototyping

Before committing to a PCB redesign, these dev boards allow testing the FPGA and toolchain:

| Board | FPGA | Price | Notes |
|-------|------|-------|-------|
| [Lattice ECP5-5G Versa](https://www.latticesemi.com/products/developmentboardsandkits/ecp5-5g-versa-development-kit) | LFE5UM5G-45F | ~$150 | Official Lattice board, well-supported in open-source flow |
| [ULX3S](https://www.crowdsupply.com/radiona/ulx3s) | LFE5U-12F/25F/45F/85F | ~$100 | Community favorite for open-source ECP5 development |
| [OrangeCrab](https://groupgets.com/manufacturers/good-stuff-department/products/orangecrab) | LFE5U-25F | ~$50 | Feather form factor, compact |
| [Sipeed Tang Nano 9K](https://wiki.sipeed.com/hardware/en/tang/Tang-Nano-9K/Nano-9K.html) | GW1NR-9 (QN88) | ~$15 | Cheapest way to test Gowin + Apicula |
| [Sipeed Tang Nano 20K](https://wiki.sipeed.com/hardware/en/tang/tang-nano-20k/nano-20k.html) | GW2AR-18 (QN88) | ~$25 | Larger Gowin option |
| [Cologne Chip GateMate Eval](https://colognechip.com/programmable-logic/gatemate-evaluation-board/) | CCGM1A1 | ~$244 | Official eval board |
| [Olimex GateMate EVB](https://www.olimex.com/Products/FPGA/GateMate/) | CCGM1A1 | ~$50 | Budget GateMate board |

---

## Toolchain Comparison for Claude Code Integration

A key requirement is the ability to iterate on FPGA designs using Claude Code without manual IDE interaction.

| Criterion | ECP5 (Yosys+nextpnr) | Gowin (Apicula) | Spartan-7 (Vivado) | GateMate (OSS) |
|-----------|----------------------|-----------------|--------------------|-----------------|
| Fully CLI-driven | Yes | Yes | Yes (TCL) | Yes |
| No license server | Yes | Yes | No (node-locked) | Yes |
| Install size | ~500 MB | ~500 MB | ~80-100 GB | ~500 MB |
| Open source | Yes (ISC/MIT) | Yes (Apache 2.0) | No (proprietary) | Yes |
| Docker-friendly | Yes | Yes | Difficult | Yes |
| Build time (~10K LUT design) | ~30 sec | ~30 sec | ~2-5 min | ~30 sec |
| VHDL support | Experimental (ghdl plugin) | Experimental (ghdl plugin) | Native | Experimental (ghdl plugin) |
| Verilog support | Native | Native | Native | Native |
| Error messages | Good | Adequate | Excellent | Adequate |
| Community examples | Many | Growing | Extensive | Few |

**For Claude Code specifically**, the open-source tools (ECP5, Gowin, GateMate) are ideal because:
- No license management or authentication barriers
- Fast iteration cycle (~30 seconds synthesis + PnR)
- Small install footprint fits in CI/CD containers
- Clear, parseable error messages from Yosys and nextpnr
- Makefile-based flow integrates naturally with standard development workflows

---

## Sources

- [Yosys Open Synthesis Suite](https://github.com/YosysHQ/yosys)
- [nextpnr — Portable FPGA Place and Route](https://github.com/YosysHQ/nextpnr)
- [Project Trellis — ECP5 Bitstream Documentation](https://github.com/YosysHQ/prjtrellis)
- [OSS CAD Suite](https://github.com/YosysHQ/oss-cad-suite-build)
- [ghdl-yosys-plugin — VHDL Synthesis](https://github.com/ghdl/ghdl-yosys-plugin)
- [Project Apicula — Gowin Bitstream Documentation](https://github.com/YosysHQ/apicula)
- [openXC7 — Open-Source Xilinx 7-Series Toolchain](https://github.com/openXC7/toolchain-installer)
- [Lattice ECP5 Family Data Sheet (FPGA-DS-02012)](https://www.latticesemi.com/-/media/LatticeSemi/Documents/DataSheets/ECP5/FPGA-DS-02012-1-9-ECP5-ECP5G-Family-Data-Sheet.ashx)
- [AMD Spartan-7 FPGAs](https://www.amd.com/en/products/adaptive-socs-and-fpgas/fpga/spartan-7.html)
- [Cologne Chip GateMate](https://colognechip.com/programmable-logic/gatemate/)
- [Gowin Semiconductor — GW1NR Series](https://www.gowinsemi.com/en/product/detail/49/)
- [Vivado TCL Batch Mode (UG835)](https://docs.amd.com/r/en-US/ug835-vivado-tcl-commands/Tcl-Batch-Mode)
- [Intel Quartus Prime Lite](https://www.intel.com/content/www/us/en/software-kit/868560/intel-quartus-prime-lite-edition-design-software-version-25-1-for-linux.html)
- [GateMate at CERN FDF 2025](https://indico.cern.ch/event/1587509/contributions/6690211/attachments/3140818/5574499/gatemate-fdf2025.pdf)
