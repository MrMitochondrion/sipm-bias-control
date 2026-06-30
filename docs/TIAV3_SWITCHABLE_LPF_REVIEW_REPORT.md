# TIAv3 Switchable LPF — Integrated Design Review Report

**Project:** `tiav3_SwitchableLPF` (v11.1) vs. reference `tiav3_s14420`
**Date:** 2026-06-29
**Scope:** Schematic + PCB review of prototype adding a digitally switchable 3-way output LC low-pass filter to an existing SiPM transimpedance amplifier design.
**KiCad version:** 9.0 (schema format 20250114 / PCB format 20241229)
**Method:** 6-agent parallel analysis — repository structure, original circuit, new schematic, mux architecture, filter/signal integrity, PCB/testing.

> **Constraint:** This is a read-only review. No schematic or PCB design files were modified.

---

## 1. KiCad MCP / UI Status

KiCad 9.0 was running (PID 16954) at review time. However, the MCP backend reported **"No board is loaded"** — no project was open in the KiCad session. All analysis was therefore performed by directly reading `.kicad_sch` and `.kicad_pcb` source files.

MCP tools available for future interactive sessions:
- `run_erc` — automated electrical rules check
- `run_drc` — design rule check
- `list_schematic_components` / `list_schematic_nets` — netlist cross-check
- `get_board_2d_view` — visual placement confirmation
- `check_courtyard_overlaps` — critical for this dense board

---

## 2. Repository and Project Structure

```
sipm-bias-control/              ← git repository root
├── tiav3_s14420/               # Reference design (frozen)
│   ├── tiav3.kicad_sch         # 5702 lines, A4 sheet
│   ├── tiav3.kicad_pcb         # ~7449 lines, circular board
│   ├── tiav3.kicad_pro / .prl
│   ├── tiav3-backups/          # 1 zip (2026-06-29, end-of-day snapshot)
│   ├── tiav3                   # ⚠ Stale tiav2 BOM CSV (misnamed, see below)
│   ├── tiav3.jpg               # Board photo
│   ├── bom/ibom.html
│   ├── footprints.pretty/      # Custom SiPM and OPA847 footprints
│   └── schlib/ + sym-lib-table # Legacy KiCad 5.x .lib format
│
├── tiav3_SwitchableLPF/        # New prototype (active dev, 2026-06-29)
│   ├── tiav3.kicad_sch         # 10066 lines, A3 sheet (~1.77× original)
│   ├── tiav3.kicad_pcb         # ~13201 lines, circular board
│   ├── tiav3.kicad_pro / .prl
│   ├── tiav3-backups/          # 5 zips (all 2026-06-29, active session)
│   ├── Digital Switch/         # Downloaded vendor assets
│   │   ├── TMUX1209RSVR/KiCADv6/  ← symbol + footprint
│   │   └── SSQ-102-01-G-D/        ← symbol + footprint
│   ├── fp-info-cache           # KiCad-generated cache (gitignore-able)
│   ├── tiav3.jpg               # Same board photo (from s14420)
│   ├── footprints.pretty/      # Identical to s14420/
│   └── schlib/ + sym-lib-table # Identical to s14420/ (legacy .lib format)
│
├── obsolete/                   # tiav2, tia-microfj, etc.
├── bias-control-lt8362/
├── power-supply/
└── ...
```

### Derivation Evidence

`tiav3_SwitchableLPF/` was created as a direct folder copy of `tiav3_s14420/` in git commit `3dde3ef` (2026-06-29 15:08): *"Created digitally switchable LPF SiPM based on tiav_14420"*. The schematic UUID (`548ba79d-aa60-452e-83bc-59e3672545ef`) is shared between both files — the new schematic was edited in-place rather than started fresh.

### Structure Red Flags

| Item | Severity | Notes |
|------|----------|-------|
| TMUX1209 footprint not in `fp-lib-table` | **High** | `RSV0016A.kicad_mod` is in `Digital Switch/TMUX1209RSVR/KiCADv6/footprints.pretty/` but the project `fp-lib-table` only points to `${KIPRJMOD}/footprints.pretty`. PCB netlist sync will report footprint-not-found. |
| SSQ-102-01-G-D footprint not in `fp-lib-table` | **Medium** | Same issue — footprint in `Digital Switch/SSQ-102-01-G-D/` but not registered. |
| `.DS_Store` files committed | Low | 3 `.DS_Store` files committed; no `.gitignore` in repo. |
| Shared schematic UUID | Low | Both schematics share UUID `548ba79d-...`; won't cause functional issues but confuses some tools. |
| `tiav3` unnamed file in `tiav3_s14420/` | Low | This is a stale BOM CSV from `tiav2` (Eeschema 5.99, 2021). Should be renamed or deleted. |
| Legacy `.lib` schlib format | Low | Both projects use KiCad 5.x `.lib` files; works with KiCad 9 compat mode but should be migrated. |
| No README or design notes | Low | Neither project folder has documentation on design intent, supply voltages, or SiPM model assumptions. |

---

## 3. Design Comparison: tiav3_s14420 → tiav3_SwitchableLPF

| Attribute | tiav3_s14420 | tiav3_SwitchableLPF |
|-----------|-------------|---------------------|
| Sheet size | A4 | A3 |
| Schematic lines | 5702 | 10066 |
| PCB lines | ~7449 | ~13201 |
| OPA847 TIA | U1 | U1 (preserved) |
| OPA842 buffer | None | **U2 (new)** |
| TMUX1209 mux | None | **U3 (new)** |
| Digital control header | None | **J3 SSQ-102-01-G-D (new)** |
| Output LPF | Fixed 10 MHz CLC | **Switchable 10/15/20 MHz + bypass** |
| Output series resistor | No (50Ω externally terminated) | **R1=50Ω series at DA input** |
| Component count (approx.) | ~21 | ~37 (nearly doubled) |
| Board outline | Circle, ∅25.35 mm | Circle, ∅25.35 mm (unchanged) |
| Stackup | 4-layer FR4 (JLC7628) | 4-layer FR4 (JLC7628, identical) |
| Board silkscreen | (unlabeled) | "Digital Switchable LPF SiPM TIAv11.1" |

**Preserved without change:** D1 (S14420 SiPM), U1 (OPA847IDBVT), J1 (SSQ-103 6-pin bias/power header), J2 (SMB output), R2 (1.5 kΩ TIA feedback), R3 (68 Ω offset), R5 (3.3 Ω input damp), R7 (5 kΩ offset), R11 (10 Ω HV damp), R23 (33 Ω pole-zero), L1 (100 nH pole-zero), C1/C4 (2.2 µF 100 V bias), C5/C7/C8/C9 (op-amp decoupling).

---

## 4. Original tiav3_s14420 Circuit Summary

### Complete Component List

| Ref | Value | Package | Function |
|-----|-------|---------|----------|
| D1 | S14420 SiPM | Custom S14420 | Silicon photomultiplier detector |
| U1 | OPA847IDBVT | SOT-23-6 (DBV6) | Transimpedance amplifier, 3.9 GHz GBW |
| J1 | SSQ-103-01-G-D | 2×3 pin header | Bias, power (VDDA/VSSA/+VDC/GND/LINE) |
| J2 | Conn_Coaxial | SMB Jack Vertical | Signal output, 50 Ω |
| R1 | 50 Ω | 0402 | Output series termination |
| R2 | 1.5 kΩ | 0402 | TIA feedback resistor (transimpedance) |
| R3 | 68 Ω | 0402 | Offset reference lower divider leg |
| R5 | 3.3 Ω | 0402 | Pole-zero input damping |
| R7 | 5 kΩ | 0402 | Offset reference upper divider leg |
| R11 | 10 Ω | 0402 | HV bias damping resistor |
| R23 | 33 Ω | 0402 | Input series resistor (pole-zero) |
| L1 | 100 nH | 0603 | Pole-zero inductor |
| L2 | 1500 nH | 0603 | LPF series inductor (10 MHz filter) |
| C1/C4 | 2.2 µF / 100 V | 1206 | HV bias shunt decoupling |
| C2/C3 | 330 pF | 0402 | LPF shunt capacitors |
| C5 | 1 µF / 10 V | 0402 | VDDA supply decoupling |
| C7 | 1 µF | 0402 | Offset reference bypass |
| C8 | 22 µF | 0603 | Offset reference bulk bypass |
| C9 | 1 µF / 10 V | 0402 | VSSA supply decoupling |

### Signal Chain

```
+VDC → R11(10Ω) ──────────────────────────────────┐
                 ↓                                 │
            C1/C4(2.2µF/100V, shunt to GND)       │
                 ↓ VBiasFilt                       │
         D1 cathode (SiPM K pin)                   │
         D1 anode → R23(33Ω) → U1 pin4 (−)        │
                                    ↑              │
              R5(3.3Ω)+L1(100nH)                   │
              shunt to GND [pole-zero]             │
                                                   │
         U1 (OPA847) TIA:                          │
           (+) ← R7(5kΩ)/R3(68Ω) divider from LINE (DAC)
           (−) ← SiPM anode via R23/R5/L1 network
           Feedback: R2(1.5kΩ) from output to (−)
           DIS (pin5): no-connect [always enabled]

         U1 output → R1(50Ω) → LPF input
                         ↓
         C2(330pF, shunt) → L2(1500nH, series) → C3(330pF, shunt) → J2 (SMB)
```

### Key Parameters

- **Transimpedance gain:** R2 = 1.5 kΩ (V/A)
- **Pole-zero filter:** R23=33 Ω input series + R5=3.3 Ω || L1=100 nH shunt. Zero at f ≈ R23/(2πL1) ≈ 52 MHz; compensates SiPM capacitance loading.
- **DC offset adjustment:** DAC (LINE net) through R7/R3 divider to OPA847(+). C7+C8 bypass the reference node to suppress DAC noise.
- **Output LPF:** 3rd-order Butterworth C-L-C, C2=C3=330 pF, L2=1500 nH → f₋₃dB ≈ 10 MHz into 50 Ω. Designer documented alternatives: 8–50 MHz range.
- **Load assumption:** 50 Ω terminated instrument at J2.
- **Source assumption:** SiPM acts as current source with significant parallel capacitance (S14420 ≈ 50–200 pF depending on cell count/bias).

---

## 5. New tiav3_SwitchableLPF Schematic Review

### Full Signal Chain

```
SiPM(D1) → R23(33Ω) → U1(OPA847 TIA, Rf=R2=1.5kΩ)
   ↑ offset via R7/R3 from LINE                   ↓
   ↑ pole-zero: R5(3.3Ω)/L1(100nH)         OUT net
                                                   ↓
                                     R4(200Ω) series
                                                   ↓
                                     U2 (OPA842, inverting)
                                       R6(200Ω) to GND at (+)
                                       R8(400Ω) feedback
                                       Gain = −R8/R4 = −2
                                                   ↓
                                     C6(0.1µF decoupling)
                                                   ↓
                                     R1(50Ω) → DA (TMUX1209 A-side common)
                                                   ↓
             ┌──────────────────┬──────────────────┬──────────────────┐
             S1A               S2A               S3A               S4A
          10 MHz LPF        15 MHz LPF        20 MHz LPF        (bypass)
          C2/L2/C3          C13/L3/C14        C15/L4/C16        (short)
             S1B               S2B               S3B               S4B
             └──────────────────┴──────────────────┴──────────────────┘
                                                   ↓
                                           DB (TMUX1209 B-side common)
                                                   ↓
                                                  J2 (SMB output)
```

### New Components

**U2 — OPA842xDBV (SOT-23-5), Inverting Buffer**
- Non-inverting input (+): GND via R6=200 Ω (bias)
- Inverting input (−): OUT net through R4=200 Ω
- Feedback: R8=400 Ω
- **Actual gain: −400/200 = −2** (schematic labels it "Gain of −1" — incorrect annotation)
- Supply: VDDA/VSSA (same analog rails as OPA847)
- Decoupling: C6=0.1 µF at output node
- Bandwidth at gain −2: ~100 MHz (OPA842 GBW=200 MHz, unity-gain stable)
- Purpose: isolates OPA847 from TMUX1209 input capacitance; provides signal inversion and 2× amplification

**U3 — TMUX1209RSVR (WQFN-16 / RSV0016A)**
- DA (pin 6): signal input from U2 output (via R1=50 Ω)
- DB (pin 7): signal output → J2
- S1A–S4A (pins 2–5): filter A-side inputs
- S1B–S4B (pins 11–8): filter B-side outputs
- VDD (pin 12): VDDA with C12=1 µF decoupling
- GND (pin 13): GND (digital ground)
- A0 (pin 15), A1 (pin 14), EN (pin 16): digital control from J3
- NC (pin 1): no-connect marker present

**J3 — SSQ-102-01-G-D (Samtec 2×2 socket)**
- Pin 01: A0, Pin 02: A1, Pin 03: EN, Pin 04: DGND
- No series protection resistors on control lines
- No pull-down on EN — floats if J3 is unpopulated

### Filter Networks (Confirmed Component Values)

All three filters are 3rd-order Butterworth C-L-C (Pi topology) designed for 50 Ω source and 50 Ω load:

| Channel | A0 | A1 | C_in | L_series | C_out | f₋₃dB (50Ω ideal) |
|---------|----|----|------|----------|-------|-------------------|
| S1A/S1B | 0 | 0 | C2=330 pF | L2=1500 nH | C3=330 pF | **10 MHz** |
| S2A/S2B | 0 | 1 | C13=220 pF | L3=1000 nH | C14=220 pF | **15 MHz** |
| S3A/S3B | 1 | 0 | C15=160 pF | L4=820 nH | C16=160 pF | **20 MHz** |
| S4A/S4B | 1 | 1 | (none) | (none) | (none) | **Bypass / wideband** |

Filter math verification:
- 10 MHz: f = 1/(2π√(1500nH × 165pF)) ≈ **10.1 MHz** ✓
- 15 MHz: f = 1/(2π√(1000nH × 110pF)) ≈ **15.2 MHz** ✓
- 20 MHz: f = 1/(2π√(820nH × 80pF)) ≈ **19.6 MHz** ✓

All three match design intent within component tolerances.

### Mux Truth Table

```
EN   A0   A1   Active path
─────────────────────────────
0    x    x    All channels off (DB = high-Z)
1    0    0    S1A/S1B → 10 MHz filter
1    0    1    S2A/S2B → 15 MHz filter
1    1    0    S3A/S3B → 20 MHz filter
1    1    1    S4A/S4B → bypass (no filter)
```

*Note: A0/A1 bit ordering in this truth table is from schematic annotation; verify against TMUX1209 datasheet (standard truth table uses A1 as MSB). Both orderings produce the same channel 1 (00→10MHz) and channel 4 (11→bypass) assignments.*

---

## 6. Mux/Filter Switching Architecture Review

### Architecture Classification

The TMUX1209RSVR switches **both** the input side (DA → SxA) and the output side (SxB → DB) of each filter path simultaneously. This is **Architecture C** — correct and necessary for a 2-terminal passive Pi filter with a terminated source and terminated load.

Key implications:
- Inactive filter shunt capacitors are disconnected at **both** ends — they cannot load the active signal path through direct DC connection.
- Only the off-state switch capacitance (Coff ≈ 18 pF per switch) couples inactive paths to the active path.
- This is the correct topology. A single-sided switch (input only or output only) would leave the other end of each inactive filter floating or connected, causing resonance problems.

### Isolation of Inactive Channels

With 3 inactive channels, each S_xA and S_xB pin presents Coff ≈ 18 pF to the DA and DB nodes respectively:

- Total parasitic capacitance at DA: 3 × 18 pF = **54 pF**
- Total parasitic capacitance at DB: 3 × 18 pF = **54 pF**
- Parasitic pole at DA: f_p = 1/(2π × 50 Ω × 54 pF) ≈ **59 MHz**
- This is above all three filter cutoffs — the passband is unaffected, but the 59 MHz parasitic limits bandwidth in the bypass (S4) mode.

### Control Interface Analysis

- A0, A1, EN route through net labels from J3 (Samtec SSQ-102-01-G-D connector)
- No series resistors on control lines (ESD/ringing protection)
- **EN has no pull-down resistor** — if J3 is unpopulated or floating, EN is undefined; TMUX1209 may enable a random channel
- DGND appears as a separate net on J3 pin 04; its connection to the main GND plane is unclear in the schematic

### Power Handling

- VDD decoupling: C12=1 µF at TMUX1209 pin 12 ✓
- GND connection at pin 13 ✓
- NC pin 1 properly marked no-connect ✓
- EN is active HIGH per schematic annotation

---

## 7. Filter and Signal Integrity Review

### Impedance Environment

The complete output chain from OPA842 to J2 is designed as a 50 Ω system:

| Point | Impedance | Source |
|-------|-----------|--------|
| OPA842 Zout | < 5 Ω closed-loop | OPA842 spec |
| R1 (series) | 50 Ω | Sets 50 Ω source Z for filters |
| DA node effective Z | ~50–54 Ω | R1 + TMUX Ron ≈ 4 Ω |
| Each C-L-C filter | Designed for 50 Ω in/out | Component values confirmed |
| DB node | ~4 Ω (TMUX Ron) + filter output Z | Low at DC, 50 Ω at passband edge |
| J2 (SMB out) | — | Must be externally 50 Ω terminated |

**50 Ω external termination at J2 is mandatory.** Without it, all three C-L-C Pi filters become mismatched and will ring at their natural resonance (e.g., ~17 MHz for the 10 MHz filter LC pair). Connecting a 1 MΩ oscilloscope probe without a 50 Ω adapter will produce severe ringing that looks like a circuit fault.

### Impact of TMUX1209 Ron on Filter Response

TMUX1209 Ron ≈ 4 Ω typical (both active switch legs):
- Effective source impedance: R1 + Ron_A ≈ 54 Ω (8% above 50 Ω design)
- Effective load impedance: Ron_B in series with 50 Ω = 54 Ω at DB
- Filter cutoff shifts ~5–8% downward (toward ~9.2, 13.8, 18.4 MHz)
- This shift is within acceptable tolerance for a prototype

### Coff Loading on DA and DB Nodes

Each inactive S_xA pin contributes Coff ≈ 18 pF to DA. With 3 inactive channels:
- DA parasitic: ~54 pF, parasitic pole ≈ 59 MHz
- DB parasitic: ~54 pF, parasitic pole ≈ 59 MHz
- Effect on S4 (bypass) path: bandwidth limited to ~59 MHz through this mechanism — acceptable for SiPM pulse observation

### Inductor SRF Concern

- L2 = 1500 nH (0603): typical SRF > 200 MHz — well above 10 MHz cutoff ✓
- L3 = 1000 nH (0603): typical SRF ~150–200 MHz — well above 15 MHz cutoff ✓
- L4 = 820 nH (0603): typical SRF ~100–150 MHz — still above 20 MHz cutoff ✓
- Verify SRF ≥ 5× cutoff frequency for selected parts. DCR (0.3–1 Ω typical) adds minor passband loss.

### Signal Swing vs. TMUX Supply (Critical)

The TMUX1209 requires the analog signal at DA/DB to remain within **GND ≤ V_signal ≤ VDD**. The OPA842 is powered from VDDA/VSSA (bipolar supply) and can produce negative output voltages. If SiPM pulses produce negative swings at the OPA842 output:
- The TMUX1209 will clamp negative excursions at GND (0 V)
- This causes waveform clipping, which may be misidentified as TIA saturation or ADC clipping
- Mitigation: ensure the signal DC level after OPA842 keeps the full pulse swing within [0 V, VDDA]

---

## 8. PCB Layout Risk Review

### Board Overview

| Property | Value |
|----------|-------|
| Shape | Circular |
| Diameter | 25.35 mm (~1 inch) |
| Layers | 4 (F.Cu signal / In1.Cu GND / In2.Cu PWR / B.Cu signal) |
| Fab spec | JLC7628 prepreg (controlled impedance, 50 Ω microstrip) |
| Silkscreen | "Digital Switchable LPF SiPM TIAv11.1" |
| Approximate 50Ω trace width | ~0.2 mm on F.Cu over In1.Cu (GND) for JLC7628 |

### Key Placement (Extracted from PCB File)

| Component | Location (mm) | Notes |
|-----------|--------------|-------|
| D1 (SiPM) | (200, 100), B.Cu back | Board center, bottom side |
| U1 (OPA847) | (192.6, 99.2), F.Cu | 7.4 mm from SiPM |
| U2 (OPA842) | (194.7, 106.7), F.Cu | 4.6 mm from U1 |
| U3 (TMUX1209) | (193.3, 93.4), F.Cu | 13.4 mm hop from U2 |
| J3 (SSQ-102) | (190.9, 100.4) | Near board center-left |
| J2 (SMB out) | (200.0, 91.9) | Top edge |
| J1 (bias header) | (206.9, 97.5) | Right edge |

**U2 → U3 inter-stage distance = 13.4 mm.** On a 25 mm board with 200 MHz signals, this trace length is borderline for uncontrolled impedance stubs. A 13.4 mm trace at 200 MHz ≈ λ/75 — reflections will exist if not controlled to 50 Ω. This is the primary layout risk.

### Top 5 Layout Risks

1. **U2 → U3 signal trace (13.4 mm, High).** Trace must be routed as controlled-impedance microstrip (~0.2 mm width on F.Cu over In1.Cu). Any wide meander, via, or uncontrolled segment will cause reflections and frequency response ripple. This is the most critical routing task.

2. **OPA847 TIA feedback parasitics (High).** R2 (1.5 kΩ) and any Cf must be within 0.5–1 mm of U1 pins 1 and 4. The 3.9 GHz GBW OPA847 is hypersensitive to stray capacitance at the inverting input. Avoid copper fill on F.Cu under the feedback network. Use a keepout zone.

3. **DGND-to-analog GND coupling (Medium-High).** If DGND is not star-connected to analog GND at a single quiet point, switching transients on A0/A1/EN will inject into the TIA or mux signal path. Route digital control traces (J3 to U3) as a bundle away from the U1→U2→U3 signal chain.

4. **TMUX1209 VDD decoupling placement (Medium).** C12 (1 µF) must be within 0.5 mm of U3 pin 12 (VDD). On a dense 25 mm board, this may compete with other components for space. Verify courtyard clearance.

5. **HV bias routing (+VDC) near TIA input (Medium).** The SiPM bias voltage (25–60 V) traces from J1 must not run adjacent to the OPA847 inverting input or the U2 input. Cross-coupling from HV transients into the TIA summing node will add noise. Route HV on B.Cu if possible, away from F.Cu signal traces.

### Recommended Placement Priority

1. SiPM D1 (fixed at center, B.Cu)
2. OPA847 U1 — as close to SiPM anode pad as geometry allows
3. R2 (TIA feedback) and any Cf — within 0.5 mm of U1 pins 1 and 4; keepout zone for copper fill
4. OPA847 supply decoupling (C5, C9) — adjacent to U1 pins 6 and 2
5. OPA842 U2 — immediately downstream of U1, short signal trace
6. OPA842 supply decoupling (C6) — adjacent to U2
7. TMUX1209 U3 — near J2 (SMB output), controlled-impedance trace from U2
8. C12 (TMUX VDD decoupling) — adjacent to U3 pin 12
9. Filter LC networks (L2-L4, filter caps) — between U3 and J2, GND vias under each shunt cap
10. J3 digital control — board edge, away from signal chain; route A0/A1/EN as bundle

### Ground Plane Strategy

- **In1.Cu (inner GND):** Keep solid and unbroken. Do not remove copper under TIA feedback network — the GND reference there should be continuous.
- **F.Cu GND fill:** Use GND copper fill on F.Cu except under the OPA847 feedback network (keepout zone) and under high-speed signal traces where it would create unwanted capacitance.
- **DGND → GND star connection:** Single via cluster near J3, away from TIA. Do not create a separate DGND island that floats.
- **Via stitching:** Place stitching vias around U1 and along the boundary between digital control traces and the analog signal path.
- **In2.Cu (inner PWR):** Separate VDDA and VSSA poured regions. Confirm both connect to J1 supply pins via low-impedance paths.

---

## 9. Things That Look Correct

- **Filter math is verified.** All three C-L-C Pi filters produce the correct 3rd-order Butterworth −3 dB frequencies within 2% of nominal (10/15/20 MHz into 50 Ω). The component values are consistent with the Marki Microwave tool table documented in the original schematic.
- **Mux topology is correct.** Switching both DA (input) and DB (output) simultaneously maintains signal continuity and prevents floating filter nodes. This is the right way to switch 2-terminal passive filters.
- **OPA842 buffers OPA847 from TMUX capacitance.** The OPA847 (decompensated, 3.9 GHz GBW, capacitance-sensitive) never sees the TMUX1209 input capacitance directly — OPA842 provides isolation. This is a critical architectural decision.
- **R4=200 Ω input series.** Isolates OPA847 output from OPA842 input capacitance. Correct practice for driving a decompensated op-amp into a high-GBW stage.
- **R1=50 Ω source termination.** Sets correct source impedance for the C-L-C filters.
- **C12=1 µF TMUX VDD decoupling is present.**
- **TMUX NC pin (pin 1) has no-connect marker applied.** Correct.
- **4-layer stackup with dedicated GND inner plane** is appropriate for this frequency range.
- **JLC7628 prepreg specification** enables controlled impedance fabrication at JLCPCB without custom stackup requests.
- **Pole-zero network preserved** from original — correct for the S14420 SiPM.
- **DC offset adjustment preserved** — essential for operating point control.
- **Five timestamped backups** during active development — good practice.
- **S4 bypass channel** is a useful feature for v1 baseline and wideband measurements.

---

## 10. Likely Issues and Risks

### Risk 1 — TMUX1209 VDD: VDDA Must Be ≤ 5.5 V
**Severity: HIGH — go/no-go before ordering**

TMUX1209RSVR absolute maximum VDD = 5.5 V. The schematic connects TMUX1209 VDD to VDDA (the same positive analog supply as OPA847 and OPA842). If VDDA = +5 V the chip is safe. If VDDA = +12 V or +15 V (common for OPA847 operation), the TMUX1209 will be destroyed on first power-up.

The schematic does not annotate the VDDA voltage anywhere. This must be resolved before ordering.

- If VDDA = +5 V: no change needed.
- If VDDA > 5.5 V: add a 3.3 V or 5 V LDO regulator for TMUX1209 VDD, separate from the op-amp supply.

### Risk 2 — OPA842 Output Signal Swing Into TMUX1209 (Negative Clipping)
**Severity: Medium-High**

TMUX1209 requires the analog signal to stay within **0 V (GND) ≤ V_signal ≤ VDD (VDDA)**. OPA842 is powered from bipolar VDDA/VSSA and can produce negative output. If any SiPM pulse causes the OPA842 output to go below 0 V, the TMUX1209 will clamp it.

Mitigation options:
- Ensure the OPA842 output DC level (set by the offset circuit) keeps all signal excursions positive, OR
- Use TMUX1209 with a dual supply (tie VSS to VSSA), OR
- AC-couple the OPA842 output into the mux with a DC block capacitor (loses DC offset information)

### Risk 3 — OPA842 Gain Annotation Error
**Severity: Medium (schematic bug, no PCB impact)**

The schematic labels U2 as "Inverting Amplifier with Gain of −1" but the component values give:
- Gain = −R8/R4 = −400 Ω / 200 Ω = **−2**

If gain = −1 was intended: change R8 to 200 Ω.
If gain = −2 was intended: fix the schematic annotation.

This matters for signal headroom. At gain = −2, the OPA842 output swing is 2× the TIA output. Verify this does not cause saturation for the largest expected SiPM pulses, and that the TMUX1209 input swing limit (see Risk 2) is respected.

###
### Risk 4 — EN Pin Floating When J3 Unpopulated
###
**Severity: Medium**

No pull-down or pull-up resistor on the EN pin. If J3 is not populated or the control connector is disconnected, EN floats and the TMUX1209 may enable any channel. Add a 10 kΩ pull-down from EN to GND to ensure default-off behavior.

### Risk 5 — TMUX1209 Footprint Not in Project Library
**Severity: Medium (will block PCB sync)**

`RSV0016A.kicad_mod` (TMUX1209 footprint) is in `Digital Switch/TMUX1209RSVR/KiCADv6/footprints.pretty/` but not registered in the project `fp-lib-table`. PCB netlist sync will fail with "footprint not found." The footprint must be either:
- Copied to `tiav3_SwitchableLPF/footprints.pretty/`, or
- Registered in `fp-lib-table` with the correct path

Similarly for the SSQ-102-01-G-D footprint.

### Risk 6 — S4 Bypass: DB Floats When Selected
**Severity: Low-Medium**

When A0=1, A1=1, EN=1, the mux connects S4A and S4B which have no components. The DB output floats, causing undefined signal at J2. This state should be avoided in firmware/MCU control logic. If a wideband bypass path is desired, place a 0 Ω jumper or short between S4A and S4B pads.

### Risk 7 — DGND Power Symbol Type Mismatch
**Severity: Low (schematic quality, ERC warning)**

J3 pin 04 (DGND) uses a `power:VDDA`-type symbol with the value changed to "DGND". KiCad ERC will flag a power symbol type mismatch. Functionally, DGND connects to GND (confirmed by net trace), but the symbol type causes ERC noise and could confuse future editors.

Fix: replace with a proper `power:GND` symbol and rename the net to DGND if a separate digital ground domain is required.

---

## 11. Schematic TODOs

- [ ] **Resolve VDDA voltage.** Add a schematic annotation or power symbol note specifying VDDA = +X V. Confirm VDDA ≤ 5.5 V for TMUX1209 compatibility.
- [ ] **Fix OPA842 gain annotation.** Change label from "Gain of −1" to "Gain of −2" OR change R8 from 400 Ω to 200 Ω to achieve actual gain = −1.
- [ ] **Add EN pull-down.** Add a 10 kΩ resistor from EN (U3 pin 16) to GND on the schematic.
- [ ] **Fix DGND power symbol type.** Replace the mistyped VDDA symbol on J3 pin 04 with a proper GND-type or dedicated DGND symbol.
- [ ] **Document S4 path.** Add schematic text: "S4A/S4B: wideband bypass (no filter). Place 0Ω jumper between S4A–S4B if needed. Do not select A0=A1=1 while EN=1 without S4A/S4B populated."
- [ ] **Add supply voltage annotations.** Document recommended operating voltages: VDDA = ?, VSSA = ?, +VDC = ?, +5V = ? (for TMUX control logic). Essential for anyone bringing up the board.
- [ ] **Resolve OPA842 variant.** The 'x' in OPA842xDBV should be resolved to a specific suffix (recommend OPA842IDBVT for −40 to +125 °C industrial range).
- [ ] **Add series protection on J3 control lines.** Consider 33 Ω series resistors on A0, A1, EN between J3 and U3 to limit ringing and provide ESD protection.
- [ ] **Confirm TMUX1209 DGND/GND split.** Explicitly show where DGND connects to GND on the schematic; star ground at a specific via.
- [ ] **Add ERC no-connect markers** on OPA847 DIS pin if floating is intentional, and verify ERC passes after above fixes.

---

## 12. PCB Layout TODOs

- [ ] **Register TMUX1209 footprint.** Copy `RSV0016A.kicad_mod` to `footprints.pretty/` or add `Digital Switch/TMUX1209RSVR/KiCADv6/footprints.pretty/` to `fp-lib-table`. Verify pin 1 orientation matches schematic mirror.
- [ ] **Register SSQ-102-01-G-D footprint** similarly in `fp-lib-table`.
- [ ] **Run `check_courtyard_overlaps`** before sending Gerbers — with ~37 components on a 25.35 mm circular board, courtyard violations are likely.
- [ ] **Run full DRC** and resolve all violations, particularly around SMB connector pads and new IC placements.
- [ ] **Route U2→U3 signal trace as controlled-impedance microstrip.** Use ~0.2 mm width on F.Cu over In1.Cu (GND) for 50 Ω. Minimize length; avoid vias in this segment.
- [ ] **Add keepout zone on F.Cu under OPA847 feedback network** (R2, Cf if present) to prevent accidental copper fill.
- [ ] **Add test point pads** (see Section 14 for list). At minimum: TP at U1 output, U2 output, and U3 DA.
- [ ] **Verify GND via stitching** along the boundary between J3/A0/A1/EN digital traces and the U1→U2→U3 analog signal path.
- [ ] **Verify In2.Cu VDDA/VSSA pour coverage** — both analog supply planes should be solid on In2.Cu to minimize supply impedance at high frequency.
- [ ] **Check SMB connector return path.** GND shell of J2 must have multiple short vias to In1.Cu GND plane; avoid single-via return for an RF connector.
- [ ] **Route digital control traces (J3→U3) as a bundle on a single layer**, separated from signal path. Consider routing on B.Cu.
- [ ] **Verify L4=820 nH inductor part selection** — choose a part with SRF ≥ 100 MHz (≥5× cutoff).

---

## 13. Simulation TODOs

- [ ] **Full chain SPICE:** OPA847 TIA → OPA842 ×−2 → R1(50Ω) → each filter → 50Ω load. Confirm −3 dB at 10/15/20 MHz. Include OPA847 GBW rolloff and OPA842 bandwidth.
- [ ] **TMUX1209 Ron impact:** Simulate each filter with 4 Ω Ron in series at both DA and SxB. Measure −3 dB shift vs. ideal.
- [ ] **Coff parasitic pole:** Add 3 × 18 pF to DA and DB nodes; simulate S4 bypass path bandwidth with Coff loading. Confirm ≥50 MHz −3 dB in bypass mode.
- [ ] **OPA842 phase margin:** Simulate OPA842 closed-loop with R4=200 Ω, R8=400 Ω, load = TMUX1209 input impedance. Check for peaking at gain = −2.
- [ ] **OPA847 stability:** Simulate with estimated stray capacitance (0.5 pF at inverting input, 1 pF at output). Check phase margin ≥45°. Identify optimal Cf if needed.
- [ ] **Switching transient:** Simulate TMUX1209 channel-switching glitch amplitude and settling time at J2 output.
- [ ] **Monte Carlo:** Filter component tolerances ±5% (C) and ±2% (L); bound worst-case −3 dB shift.

---

## 14. Measurement TODOs

### Recommended Test Points

| TP | Net / Location | Purpose |
|----|---------------|---------|
| TP1 | U1 output (OPA847 pin 1) | TIA gain, bandwidth, DC offset |
| TP2 | U2 output (OPA842 pin 1) | Buffer stage verification |
| TP3 | U3 DA (TMUX1209 pin 6) | Signal into mux; all filter paths visible here |
| TP4 | VBiasFilt (R11→C1/C4 node) | HV bias level verification |
| TP5 | VDDA | Analog positive supply |
| TP6 | VSSA | Analog negative supply |
| TP7 | +5V | Digital supply (TMUX logic) |
| TP8 | DGND | Digital ground reference |
| TP9 | A0, A1, EN (U3 pins) | Mux control logic level verification |

### First 5 Measurements After Power-Up

1. **Supply rails with no ICs installed:** Apply power; measure VDDA, VSSA, +5V, +VDC at J1 with DMM. Confirm VDDA ≤ 5.5 V before installing U3.
2. **Quiescent supply currents:** With all ICs installed, measure VDDA and VSSA currents. Expect ~8–15 mA (OPA847 + OPA842). Significant excess indicates oscillation or solder fault.
3. **DC offset at U1 output (TP1):** Should be near 0 V; adjustable via R7/R3. Confirm no rail-to-rail clamping.
4. **TMUX control logic levels at TP9:** Verify VIH ≥ 0.7 × VDDA at A0, A1, EN. For VDDA=5 V, driving MCU at 3.3 V gives VIH=3.3 V vs. 3.5 V required — marginal; use 5 V logic or reduce VDDA to 3.3 V.
5. **TIA output noise at TP1:** 50 MHz scope bandwidth, 50 Ω termination. Expect noise floor ≤5 mV rms. More than 20 mV rms suggests TIA oscillation.

### Bring-Up Checklist (10 Steps)

1. **Visual inspection:** Verify U1, U2, U3 pin 1 orientations. OPA847 DIS (pin 5) must not have solder bridge to adjacent pads. Check for cold joints on TMUX1209 fine-pitch pads.
2. **Supply rails (no ICs):** Apply VDDA/VSSA/+5V at J1. Measure all rails. **Confirm VDDA ≤ 5.5 V before installing U3.**
3. **Quiescent current check:** Install all ICs. Measure VDDA and VSSA supply currents; check against datasheet idle values.
4. **DC operating point sweep:** Adjust LINE/DAC via J1; verify DC offset at TP1 (OPA847 out) and TP2 (OPA842 out) tracks the adjustment. Confirm no clamping.
5. **HV bias ramp:** Slowly ramp +VDC from 0 V to operating voltage. Monitor SiPM dark current via TP4. Current should be stable nA–µA in dark enclosure.
6. **Mux switching (DC):** Toggle EN=0/1 with DMM at J2; verify open (high-Z) vs. path continuity. Cycle A0/A1; verify DC path to each filter SxA node.
7. **TIA impulse response:** Inject a fast pulse (≤10 ns rise time) at SiPM input or OPA847 input. Observe TP1 on oscilloscope (50 Ω, 200 MHz bandwidth). Verify clean pulse, no sustained ringing.
8. **Frequency sweep per filter path:** Inject swept sine via 50 Ω source at TP2 or TP3. Measure at J2 (50 Ω scope input). Record −3 dB for each A0/A1 position. Expected: ~9–11 MHz, ~14–16 MHz, ~18–21 MHz.
9. **Dark count baseline:** Light-tight enclosure, operating bias. Record single-photon pulses at J2 for each filter position. Confirm SiPM photon peaks are visible and stable.
10. **Mux crosstalk check:** With S1 active (10 MHz), inject a 50 MHz signal and verify it is rejected ≥40 dB at J2 (stopband attenuation). Confirm switching does not corrupt neighboring measurement.

---

## 15. Cross-Agent Comparison and Resolved Discrepancies

### Agreements Across All Agents

- Filter values correct and verified independently by Agents 3 and 5
- TMUX1209 mux topology (both-sides switching) confirmed by Agents 3, 4, and 5
- OPA842 gain is −2 (not −1 as labeled) — confirmed by Agents 3, 4, and 5
- S4 is a bypass path with no filter components — confirmed by Agents 3, 4, and 5
- 50 Ω external termination is mandatory — confirmed by Agents 3 and 5
- TMUX VDD = VDDA risk — flagged independently by Agents 3 and 4
- Board is circular 25.35 mm, 4-layer JLC7628 — confirmed by Agents 3 and 6

### Resolved Discrepancy: A0/A1 Truth Table Ordering

Agent 5 listed S2 as (A0=1, A1=0) and S3 as (A0=0, A1=1), while Agents 3 and 4 listed the reverse. The TMUX1209RSVR standard truth table uses A1 as the MSB: binary address is (A1, A0) = 00→S1, 01→S2, 10→S3, 11→S4.

Resolving: A0=0, A1=0 → S1; A0=1, A1=0 → S2; A0=0, A1=1 → S3; A0=1, A1=1 → S4.
**Verify directly against TMUX1209RSVR datasheet truth table and the schematic annotation before coding firmware.**

### Unique Findings Per Agent

| Agent | Key Finding Not Covered Elsewhere |
|-------|-----------------------------------|
| Agent 1 | Footprint library not registered (PCB sync blocker); UUID shared between designs; stale tiav2 BOM file |
| Agent 2 | Complete original BOM with exact values; R11=10Ω HV damp; C7=1µF + C8=22µF offset bypass detail |
| Agent 3 | R6=200Ω at OPA842 non-inverting input; DGND implemented with wrong symbol type; S4 leaves DB floating |
| Agent 4 | EN pin floating risk without pull-down; DGND net vs GND connection point unclear |
| Agent 5 | Coff parasitic pole at ~59 MHz; L4=820nH SRF boundary check; Monte Carlo tolerance bounds |
| Agent 6 | U2→U3 13.4 mm inter-stage hop (highest routing risk); specific test point list; 10-step bring-up checklist |

---

## 16. Final Recommendation

**Verdict: Proceed to fabrication — but resolve two blockers and four fixables before ordering.**

The core architecture of `tiav3_SwitchableLPF` is sound:
- The switchable Pi-filter topology is correctly wired.
- The three filter cutoff frequencies (10/15/20 MHz) are correctly calculated.
- The OPA842 buffer isolation strategy is appropriate.
- The 4-layer stackup is right for this frequency range.
- The bypass S4 path adds useful flexibility for v1 testing.

### Blockers — Fix Before Ordering

| # | Issue | Action |
|---|-------|--------|
| B1 | **VDDA voltage unspecified; TMUX1209 VDD limit = 5.5 V** | Confirm VDDA = +5 V. If higher, add a 5 V or 3.3 V regulator for U3 VDD on the board. |
| B2 | **TMUX1209 footprint not in fp-lib-table** | Copy `RSV0016A.kicad_mod` to `footprints.pretty/` and register in `fp-lib-table`. Same for SSQ-102-01-G-D. |

### Fixables — Resolve Before Bring-Up

| # | Issue | Action |
|---|-------|--------|
| F1 | OPA842 gain annotation error | Fix schematic label to "Gain = −2" or change R8 to 200 Ω for actual gain = −1 |
| F2 | EN pull-down missing | Add 10 kΩ from EN to GND on schematic + PCB |
| F3 | OPA842 output swing vs. TMUX signal range | Verify signal DC operating point keeps OPA842 output ≥ 0 V at all times |
| F4 | U2→U3 routing | Route as controlled-impedance microstrip (~0.2 mm, F.Cu over In1.Cu) |

### Acceptable for v2

- DGND power symbol type fix (cosmetic ERC warning only)
- Series resistors on J3 control lines
- S4 path 0 Ω jumper (avoid A0=A1=1 in firmware for now)
- Full SPICE simulation (use measured sweep as validation)
- Full test point pad additions (probe with spring pins during first bring-up)
- Migration of legacy `.lib` schlib files to KiCad 7+ format

The 25 mm circular board with ~37 components is dense, and the U2→U3 inter-stage routing is the primary layout risk. Run courtyard DRC and controlled-impedance verification before submitting Gerbers. If the layout passes DRC, this is ready for a hand-assembled prototype run and initial characterization.

---

*Report generated 2026-06-29. All analysis performed by direct file parsing of `.kicad_sch` and `.kicad_pcb` source files. KiCad was running but had no project loaded; MCP ERC/DRC tools were not used. Findings should be cross-checked against schematic visual review in KiCad before acting on them.*
