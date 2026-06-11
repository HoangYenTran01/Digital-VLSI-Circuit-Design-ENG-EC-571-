# Configurable 6T Memory — BCAM / TCAM / SRAM with Logic-in-Memory

**Cadence Virtuoso · 28 nm PDK · Boston University EC 571 · Spring 2026**

Replicated and simulating a 2016 IEEE JSSC paper in Cadence Virtuoso —  verifying a 6T SRAM memory structure that reconfigures at runtime as a Binary CAM, Ternary CAM, or SRAM, and can perform bitwise AND / NOR operations directly inside the memory array without moving data.

> **Collaborator:** Yixuan Huang  
> **Reference:** S. Jeloka et al., "A 28 nm Configurable Memory (TCAM/BCAM/SRAM) Using Push-Rule 6T Bit Cell Enabling Logic-in-Memory," *IEEE J. Solid-State Circuits*, vol. 51, no. 4, 2016.

---

## Why This Problem Is Interesting

Conventional BCAM cells use 10 transistors; TCAM cells use 16. Both are 2–5× larger than a standard 6T SRAM cell, making large CAM arrays expensive in area and power. The key insight in Jeloka et al. is that you can repurpose the same 6T SRAM array for parallel search by changing how the wordlines and bitlines are driven — no extra transistors needed. As a bonus, the same mechanism enables logic-in-memory operations, reducing data movement in compute-heavy workloads.

---

## What We Tested

### 1 · Memory Cell

Standard 6T SRAM bit cell — two cross-coupled CMOS inverters with two NMOS access transistors. Sizing is dictated by two competing constraints:

- **Read stability:** pull-down NMOS must be stronger than the access transistor to prevent a read upset → pull-down at 135 nm, access at 90 nm
- **Write ability:** access transistor must be able to overpower the pull-up → pull-up PMOS at 90 nm minimum

### 2 · Search Operation (BCAM Mode)

BL and BLB are precharged to VDD. The search word drives one access transistor per cell; its complement drives the other. A mismatch — search bit is 1, stored bit is 0 — discharges one bitline. The sense amplifier fires when either bitline drops below V_REF = 0.8 V. A full match leaves both lines high.

### 3 · Search Data Corruption Problem & Fix

With many rows mismatching in parallel, bitlines discharge faster than normal. If they fall past the inverter switching threshold before the sense amplifier can react, a cell storing logic 1 gets overwritten through its own access transistors — silent data corruption.

**Fix:** Power the storage inverters from a reduced rail VDD_Lo = 0.5 V while bitlines stay precharged to full VDD = 1 V. The threshold voltage of these devices is ~0.335 V, so a stored logic 1 can only be overwritten when bitlines fall below 0.165 V. The sense amplifier triggers at 0.8 V and reprecharges — 5× above the danger threshold. No write ever occurs.

### 4 · Write Operation (Column-Wise, Two Cycles)

Data in this array is organized by column, so a standard row-wise SRAM write would corrupt other columns. Two cycles are used:

- **Cycle 1** — WL high for rows that should store 1; BL = 1, BLB = 0
- **Cycle 2** — WL high for rows that should store 0; BL = 0, BLB = 1

A write-assist supplies VDD_Lo only to the column being written. Unselected columns stay at full VDD, so the under-driven wordline cannot flip their bits.

### 5 · Logic-in-Memory (AND / NOR)

When a bitline is precharged and multiple wordlines are raised, the bitline discharges if *any* selected cell stores 0. This is a bitwise AND across rows. NOR follows from De Morgan's law: raise the wordline-bars instead of the wordlines, accessing the complemented data.

Simulated on a 3-row × 2-column array: 2-row AND and 3-row AND both returned correct results.

### 6 · Configurable Sense Amplifier

One sense amplifier per column handles both SRAM and BCAM modes via two control signals (`diff` / `diffb`):

| Mode | diff | diffb | Compares |
|---|---|---|---|
| BCAM search | 0 | 1 | Each bitline vs. V_REF = 0.8 V |
| SRAM read / LIM | 1 | 0 | BL vs. BLB differentially |

Key sizing decisions: precharge and sampling PMOS at 90 nm minimum; pull-down and enable NMOS at 180 nm (2×) to match standard inverter resistance. Upsizing the output buffer was the single biggest latency improvement — justified because there is only one SA per column regardless of array depth.

---

## Results

| Metric | Conventional 10T BCAM | This 6T Design |
|---|---|---|
| Transistors per bit | 10 | 6 |
| Array area | Baseline | ~2–5× smaller |
| Search delay (100 fF load) | ~1.28 ns | ~690 ps |
| Logic-in-memory | ✗ | AND, NOR |

---

## Repository

```
├── project/
│   ├── EC571_Project_Report.pdf    # Full written report
│   └── EC571_Project_Slides.pdf   # Presentation slides
├── labs/                           # Skill-building exercises (see below)
│   ├── Lab2/   #Voltage Transfer Characteristic
│   ├── Lab4/   # 3-input NAND — full design flow including layout 
│   ├── Lab5/   # NOR gate — 4 CMOS logic families benchmarked comparing statis dynamic power and delay 
│   └── Lab6/   # 6T vs. 8T SRAM sizing and stability analysis
└── cmos-sram-memory-cadence.tar.gz # Full Cadence zip folder 
```

> Extract the tarball and run `virtuoso &` in the project directory to open all schematics and layouts.

---

## Skill-Building Labs

The labs below trace the progression from basic gate design to memory circuits. Each builds a technique used directly in the project.

<details>
<summary><strong>Lab 4 — 3-Input NAND: Full Design Flow (Schematic → Layout → Post-Layout)</strong></summary>

Designed, laid out, and verified a static CMOS 3-input NAND gate. Ran the complete physical verification flow: DRC (zero violations), LVS (schematic–layout match confirmed), and PEX parasitic extraction. Compared pre- vs. post-layout timing.

**Propagation delays:**

| Scenario | Pre-layout t_pLH | Pre-layout t_pHL | Post-layout t_pLH | Post-layout t_pHL |
|---|---|---|---|---|
| All 3 inputs switch | 137 ps | 1190 ps | 139 ps | 1203 ps |
| 1 input fixed at VDD | 197 ps | 1861 ps | 199 ps | 1197 ps |
| 2 inputs fixed at VDD | 375 ps | 1179 ps | 378 ps | 1187 ps |

Parasitics added ~2 ps to rising delays and ~10 ps to falling delays. NMOS series chains are more sensitive to routing capacitance than parallel PMOS networks — a finding that directly informed sense amplifier sizing in the project.

</details>

<details>
<summary><strong>Lab 5 — NOR Gate: Four CMOS Logic Families Benchmarked</strong></summary>

Implemented the same 2-input NOR in static CMOS, footed/unfooted dynamic domino logic, DCVSL, and pass transistor logic, then measured delay and power for each.

| Logic Style | Propagation Delay | Static Power | Dynamic Power |
|---|---|---|---|
| Static CMOS | 330 ps | 2.06 nW | 148 µW |
| Footed Dynamic | 535 ps | ~1.2 nW | — |
| Unfooted Dynamic | 653 ps | 50 µW* | — |
| DCVSL | 49 ps | 3.5 nW | 113 µW |
| Pass Transistor | 2.5 ps | 163 nW | 82 µW |

*Unfooted dynamic logic leaks during precharge — the pull-down network is not isolated, so current flows continuously. Footed logic adds one transistor to fix this.

DCVSL offered the best speed-power balance. Understanding these trade-offs was essential context for choosing the configurable sense amplifier topology in the project.

</details>

<details>
<summary><strong>Lab 6 — 6T vs. 8T SRAM: Sizing Sweep for Read/Write Stability</strong></summary>

Swept access transistor W/L ratio in Spectre to find the boundaries where read upsets and write failures occur in both 6T and 8T SRAM topologies.

| Stability Metric | 6T SRAM | 8T SRAM |
|---|---|---|
| Read upset above W/L = | 32 | No read upset |
| Write fails below W/L = | 1.7 | 46 |

6T has an inherent tension: a stronger access transistor helps write but risks flipping stored data during read. 8T adds a dedicated read port (M7/M8), fully decoupling the two paths — at the cost of 2 extra transistors per cell. This trade-off is exactly why the project's 6T cell uses careful asymmetric sizing (pull-down at 135 nm, access at 90 nm) rather than minimum-size everything.

</details>

---

## Tools

| Tool | Role |
|---|---|
| Cadence Virtuoso | Schematic entry, analog layout |
| Cadence Spectre | SPICE-level transient and DC simulation |
| Calibre DRC | Design rule checking |
| Calibre LVS | Layout vs. schematic verification |
| Calibre PEX | Parasitic resistance/capacitance extraction |
| 28 nm / 90 nm CMOS PDK | Process design kit for device models |
