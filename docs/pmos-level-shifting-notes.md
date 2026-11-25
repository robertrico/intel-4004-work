# PMOS level shifting notes: RetroShield 8008 and 4004 vs. a 3.3 V ECP5

Project started inside a RetroShield study checkout and moved out on 2026-11-13. Paths such as `docs/k4004/...`, `hardware/...` and `docs/k8008/...` cited below are relative to `../retroshield-hw`; `bom/`, `datasheets/` and `reference/` are relative to this project.

Purpose: understand how the RetroShield boards in this repo interface a real
Intel 8008 and 4004 to 5 V logic, check that against a proposed FPGA interface,
and pick a starting circuit for an 8008 lockstep (shadow compare) board.

This is a read-only analysis. Nothing under `hardware/` was changed.

## 0. Sources and how much to trust each statement

| Tag | Meaning |
|---|---|
| [SCH] | Read from the schematic in this repo |
| [DS p.N] | Read from a datasheet or manual in this repo, page cited |
| [FW] | Read from the RetroShield Teensy firmware (separate repo, see below) |
| [SB] | Read from Scott Baker's `8008-super` project (separate repo, see 1.8) |
| [CALC] | My arithmetic from cited numbers |
| [INFER] | My reasoning, not stated in any document here |
| [MEM] | From memory. The document is not in this repo. Unverified |

Documents used:

| Document | Path | How it was read |
|---|---|---|
| 8008 Users Manual | `docs/k8008/8008UM.pdf` | Scanned, no text layer. Pages rendered and read visually. Citations use printed page numbers. Printed page N is PDF page N+3 up to printed page 15. After the fold-out schematics the offset changes (printed page 20 is PDF page 22), so later citations give the PDF page as well |
| 4004 datasheet | `docs/k4004/4004_Datasheet.pdf` | Pages 7 to 9 rendered and read visually (the text layer is unusable OCR). Printed page equals PDF page |
| 4004 datasheet, second scan | second scan of the Intel 4004 datasheet from [bitsavers](http://www.bitsavers.org/components/intel/MCS4/) | The same nine-page datasheet from a different, cleaner scan (catalog pages 8-15 to 8-23). Pages 7, 8 and 9 read visually and compared line by line with the repo copy. Every figure in 2.4 matches. It is the same document, so this confirms my reading, not Intel's numbers |
| MCS-4 Users Manual, Feb 73 | `docs/k4004/MCS-4_UM_Feb73.pdf` | 175 scanned pages. OCR keyword search, then PDF pages 61, 101 and 106 read. Citations use PDF page numbers. Not read cover to cover |
| CD4504B | `docs/k4004/cd4504b.pdf` | PDF pages 1 to 3 read visually |
| CD40109B | `docs/k4004/cd40109b.pdf` | Text layer, PDF pages 1 to 5 |
| ROE-0515S | `docs/k4004/boost/ROE0515S.pdf` | Text layer |
| 78L09 | `hardware/k8008/docs/78L09_sot89-3.pdf` | Text layer, keyword search only |
| RetroShield 4004 notes | `docs/k4004/k4004_notes.rtf` | Text extracted |
| RetroShield Teensy firmware | `../retroshield-teensy/t8008/` and `../retroshield-teensy/t4004/` (separate repo, added after the first pass) | Source read: `t8008_intellec_8` in full for `cpu_tick()`, start-up and pin macros, `t8008_scelbal_faster` for its timing header and the differences in `cpu_tick()`, `t4004_empty` by keyword only |
| 8008 manual, Rev 4, Nov 73 | `../retroshield-teensy/t8008/t8008_intellec_8/doc/Intel_8008_8-Bit_Parallel_Central_Processing_Unit_Rev4_Nov73.pdf` | A different, longer scan (132 pages). Pages 8 to 40 run through OCR and keyword searched only. Used to find the SIM8-01 text quoted in 1.6. Its electrical tables were not compared with the copy in this repo |
| Teensy adapter board | `hardware/teensy/teensy.sch` | Part list only |
| Scott Baker's 8008 "MasterBlaster" | `../8008-super` (separate repo, a different author's 8008 design) | See 1.8 for exactly which files were read |
| RetroShield Arduino Mega firmware | `../retroshield-arduino/` (separate repo, added last) | Contains 4004 sketches only: `k4004/archive/k4004_busicom` (Mega only, 2020) read for its clock and bus sequencing, `k4004/kt4004_busicom_v2` by keyword. There is no 8008 sketch in it |

In the repo but not read: `docs/k8008/MCS-8_test-system_manual.pdf`,
`docs/k4004/MCS4_Data_Sheet_Nov71.pdf`, `docs/k4004/4201_ 4MHz_Clock_Generator.pdf`,
`docs/k4004/k4004_cpu_timing_1pager.pdf`, `docs/k4004/measurements/*` (scope
shots, 4040), `docs/k4040/*`.

Not available: any Arduino Mega firmware for the 8008. The Teensy 8008 sketches
refuse to build for the Mega and point to the "Retroshield-Arduino" repo, but
that repo has no 8008 folder and its README does not list the 8008. As far as
these three repos show, the 8008 shield has only ever been run from a Teensy
4.1, even though the schematic title says "for Mega/Teensy". Also not
available: any BOM, any README for `hardware/k8008` or `hardware/k4004s`, and
datasheets for
74LS245, 74HCT541, 74LVC245, BAT54, MC74VHC1G04, BZB84, TXS0108 and the ECP5.

---

## 1. Intel 8008: RetroShield k8008 rev B "direct drive"

### 1.1 What is in the repo

`hardware/k8008/` holds only the rev B direct-drive design:

- `k8008_revB_directdrive.dch` (DipTrace schematic, binary)
- `k8008_revB_directdrive.dip` (DipTrace PCB, binary)
- `k8008_revB_directdrive.pdf` (one-page schematic)
- `docs/` with the 78L09 datasheet and three timing PNGs

There is no earlier revision and no non-direct-drive variant in the repo. The
net names on the 8008 side still end in `_15V` (for example `D7_15V`), which
suggests an earlier revision ran the chip from a +15 V shifted supply like the
4004 board does [INFER]. In rev B those nets are plain 0 to 5 V TTL nets.

The DipTrace files are binary, so the schematic below was read from the PDF
rendered at 400 dpi. Part numbers and values were cross-checked against the
UTF-16 strings embedded in the `.dch` file (SN74LS245DW, MC74VHC1G04DFT1G,
78L09-SOT89-3, BZB84-C5V6,215, ROE-0515S, 680, 1.1K all present).

### 1.2 Power rails [SCH]

| Rail | Value | Source |
|---|---|---|
| VCC (8008 pin 10) | +5 V | Arduino header pins 1 and 2, C1 1 uF to GND |
| VDD (8008 pin 1) | -9 V | See below |
| GND | 0 V | Shared with the Arduino. The 8008 has no ground pin |

How the -9 V is made:

- U4 is a RECOM ROE-0515S, an isolated, unregulated 5 V to 15 V DC/DC rated
  66 mA [DS ROE0515S.pdf p.1]. Input is VCC and GND. The output floats.
- U5 is a 78L09 positive regulator in SOT-89 placed across that floating 15 V:
  IN (pin 3) on the DC/DC +Vout (net `POS15V`), GND pin (pin 2) on the DC/DC
  -Vout (net `NEG15V`), OUT (pin 1) tied to system GND.
- The regulator holds OUT at 9 V above its own GND pin. Because OUT is nailed
  to system ground, the regulator's GND pin sits at -9 V. That node is VDD.
- `POS15V` ends up near +6 V relative to system ground and is not used.
- C3 1 uF across the DC/DC output, C2 1 uF across the regulator output
  (system GND to VDD).
- The schematic carries the note "This requires ISOLATED DC/DC!". A
  non-isolated converter would short through the regulator.

Current budget:

- 8008 IDD is 30 mA typical, 60 mA maximum at 25 C [DS 8008UM p.14].
- 78L09 quiescent current is 2.0 mA typical, 5.5 mA maximum, dropout 1.7 V
  typical [DS 78L09_sot89-3.pdf].
- Power LED branch is about (9 - 5.6 - 2.0) / 1.1k = 1.3 mA [CALC].
- Worst case is about 67 mA against a 66 mA module [CALC]. Typical is about
  34 mA. The design is fine at typical current and has no headroom at the
  datasheet maximum.

Sequencing: there is none. +5 V arrives with the Arduino and -9 V follows
whenever the DC/DC starts. Nothing gates VDD and nothing monitors it except
the LED.

Power LED: GND to green LED to R8 1.1k to D1 (BZB84-C5V6, one of the two
5.6 V zeners, the other cathode is unconnected) to VDD. The zener means the
LED only lights when VDD is more negative than roughly -7.5 V, so it is a
crude "negative rail is up" indicator [INFER from SCH].

### 1.3 Per-signal table [SCH]

All 8008-side connections. "Dir" is relative to the 8008.

| Signal | 8008 pin | Dir | Topology | Parts and values | MCU pin |
|---|---|---|---|---|---|
| D0 to D7 | 9 down to 2 | bidir | MCU to U2 A side. U2 B side to 680 ohm series resistor to 8008 pin. U2 /OE tied to GND (always enabled). U2 DIR from MCU. No pull-ups, no pull-downs, no diodes | U2 SN74LS245DW, R0 to R7 680 ohm 0805 | PL0 to PL7 (D49 to D42) |
| Data direction | n/a | n/a | MCU GPIO straight to U2 DIR. High means A to B, the MCU drives the 8008 | net `DATABUF_DIR_MEGA` | PG0 (D41) |
| S0 | 13 | out | 8008 pin direct to U3 A3. U3 B3 to MCU | U3 SN74LS245DW, no resistor | PC5 (D32) |
| S1 | 12 | out | Direct to U3 A2, B2 to MCU | same | PC6 (D31) |
| S2 | 11 | out | Direct to U3 A1, B1 to MCU | same | PC7 (D30) |
| SYNC | 14 | out | Direct to U3 A4, B4 to MCU | same | PC4 (D33) |
| phi1 (CLK1) | 16 | in | MCU to U3 A6, U3 B6 direct to 8008 pin | U3, no resistor, no pull-up | PC3 (D34) |
| phi2 (CLK2) | 15 | in | MCU to U3 A5, U3 B5 direct to 8008 pin | same | PC2 (D35) |
| READY | 17 | in | MCU to U3 A7, U3 B7 direct to 8008 pin | same | PC0 (D37) |
| INT | 18 | in | MCU to U3 A8, U3 B8 direct to 8008 pin | same | PC1 (D36) |

U3 has /OE tied to GND and DIR tied to VCC, so it is a fixed A to B buffer.
Four channels carry 8008 outputs to the MCU and four carry MCU outputs to the
8008.

Other parts, not connected to the 8008:

- Reset key: B3F-1000 switch from `KEY_RESET` to GND, R10 680 ohm pull-up to
  VCC, to MCU PG2 (D39). The 8008 has no reset pin. Firmware has to perform the
  INT start-up sequence itself.
- Debug LED: MCU PA7 (D29) to U6 MC74VHC1G04 inverter to red LED to R9 680 ohm
  to GND.
- No decoupling capacitors are drawn on U2, U3 or U6. The only capacitors on
  the sheet are C1, C2, C3 at 1 uF.

### 1.4 What the 8008 manual requires

DC, at VCC = +5 V +/-5 %, VDD = -9 V +/-5 % [DS 8008UM p.14]:

| Parameter | Limit | In volts at VCC = 5.0 |
|---|---|---|
| VIH, all inputs including clocks | VCC - 1.5 min, VCC + 0.3 max | 3.5 V min |
| VIL, all inputs including clocks | VDD min, VCC - 4.2 max | 0.8 V max, may go down to -9 V |
| VOH | VCC - 1.5 min at IOH = 0.2 mA | 3.5 V min |
| VOL | 0.4 V max at IOL = 0.44 mA, CL = 200 pF | No minimum is given |
| Input leakage | 10 uA max | |
| IDD | 30 mA typ, 60 mA max | |
| Absolute maximum, inputs and supply relative to VCC | +0.5 V to -20 V | |

Capacitance: input, data bus and output pins are each 5 pF typical, 10 pF
maximum [DS 8008UM p.15].

Statements from the manual that matter here:

- "All of the inputs are TTL compatible, but input pull-up resistors are
  recommended to insure proper VIH levels. All outputs are low-power TTL
  compatible." [DS 8008UM p.13]
- "During both the WAIT and STOPPED states the data bus output buffers are
  disabled and the data bus is floating." [DS 8008UM p.13]
- Output buffers (SYNC, S0 to S2, and the data bus) are two PMOS devices in
  series between VCC and VDD with the pin in the middle. Inputs (phi1, phi2,
  READY, INT) go to a gate with no DC load [DS 8008UM p.13, Figures 6 and 7].

AC [DS 8008UM p.14], 8008 column (8008-1 in brackets):

| Symbol | Parameter | Min | Max |
|---|---|---|---|
| tCY | Clock period | 2 us (1.25) | 3 us (3) |
| tR, tF | Clock rise and fall | | 50 ns |
| t phi1 | phi1 pulse width | 0.70 us (0.35) | |
| t phi2 | phi2 pulse width | 0.55 us (0.35) | |
| tD1 | phi1 falling to phi2 falling | 0.90 us | 1.1 us (1.1) |
| tD2 | phi2 falling to phi1 rising | 0.40 us (0.35) | |
| tD3 | phi1 falling to phi2 rising | 0.20 us (0.20) | |
| tDD | Data out delay, CL = 100 pF | | 1.0 us |
| tOH | Data out hold | 0.10 us | |
| tIH | Data in hold | not less than tSD | |
| tSD | SYNC out delay, CL = 100 pF | | 0.70 us |
| tS1 | State out delay, all states except T1 and T1I | | 1.1 us |
| tS2 | State out delay, T1 and T1I | | 1.0 us |
| tRW | READY pulse width during phi22 to enter T3 | 0.35 us | |
| tRD | READY delay to enter WAIT | 0.20 us | |

Timing notes [DS 8008UM p.15]: READY must be low before phi22 of T2 to
guarantee entry into WAIT. INT must not change within 200 ns of the falling
edge of phi1.

One inconsistency inside the manual: the SIM8-01 clock figure shows phi1 and
phi2 at 500 ns wide, tD1 1000 ns, tD2 500 ns, period 2 us [DS 8008UM p.20,
PDF p.22, Figure 11], while the AC table asks for a phi1 width of at least
0.70 us. I would design to the AC table. The RetroShield firmware does the
same (see 1.7).

### 1.5 How the RetroShield meets, or does not meet, those numbers

Driving the 8008 (clocks, READY, INT, and data in):

- Low level: an LS-TTL low is well under the 0.8 V limit. Fine.
- High level: the 8008 wants 3.5 V. An LS245 output is only guaranteed to
  about 2.4 V and typically sits near 3.4 V [MEM, LS245 datasheet not in
  repo]. The manual tells you to add pull-ups for exactly this reason. Rev B
  has none. So the design works on typical parts with close to zero guaranteed
  margin on VIH, including on the clocks [INFER].
- Edges: an LS245 output is far faster than the 50 ns limit into 10 pF [INFER].

Receiving from the 8008 (S0 to S2, SYNC, and data out):

- High level: 3.5 V minimum into an LS input that needs 2.0 V. Fine.
- Low level: the 8008 pulls its output toward VDD, which is -9 V, through a
  PMOS device. The manual only promises 0.4 V maximum while sinking 0.44 mA.
  An LS input sources less than that when pulled low, so the pin goes below
  ground until the LS245 input clamp diode conducts [INFER]. The typical
  sink capability is 1.6 to 1.9 mA at 0.4 V [DS 8008UM p.15, typical curve], so
  the clamp current is small. On the status lines nothing limits it except the
  8008 itself. On the data bus the 680 ohm resistors are in the path.
- So the RetroShield relies on the TTL input clamp diodes as the negative
  clamp. That is how "low-power TTL compatible" was meant to work in 1972, but
  it is not something to copy onto CMOS inputs without thought.

Data bus turnaround: U2 is always enabled and only DIR changes, so the
LS245 always drives one side or the other. When DIR is wrong for a moment, the
LS245 and the 8008 fight through 680 ohms. That resistor is the only
contention protection. The firmware knowingly accepts a short overlap at the
start of every T3 (see 1.7).

Clock rate: the manual gives a maximum clock period of 3 us, which means the
chip is dynamic and is not specified for a stopped or stretched clock. The
Teensy 4.1 firmware generates an in-spec clock of roughly 2.1 us (see 1.7). An
Arduino Mega toggling pins under program control is unlikely to hold a 3 us
period [INFER]. There is no Mega 8008 sketch in any of the three repos, so
this was probably never attempted, or never published.

### 1.6 Lockstep with a soft core

Signals the FPGA must observe:

| Signal | Why |
|---|---|
| S0, S1, S2 | The state: T1, T1I, T2, WAIT, T3, STOPPED, T4, T5. Coding as S0 S1 S2: 010 T1, 011 T1I, 001 T2, 000 WAIT, 100 T3, 110 STOPPED, 111 T4, 101 T5 [DS 8008UM p.4] |
| SYNC | Marks which of the two clock periods of a state you are in. High during the first period (phi11, phi21), low during the second (phi12, phi22) [DS 8008UM p.4 Figure 1, p.15] |
| D0 to D7 during T1 | Low 8 address bits (or register A in an I/O cycle) |
| D0 to D7 during T2 | High 6 address bits on D0 to D5 and the cycle type on D6, D7. D6 D7 = 00 PCI (instruction fetch), 01 PCR (memory read), 10 PCC (I/O), 11 PCW (memory write) [DS 8008UM p.5] |
| D0 to D7 during T3 of a PCW cycle | Write data |

Signals the FPGA must drive:

| Signal | Why |
|---|---|
| phi1, phi2 | The timebase |
| READY | Keep high for full speed, or use it to stall the real chip. It must be valid around phi22 of T2 |
| INT | Needed even if you never use interrupts. It is the only way out of the STOPPED state after power-up |
| D0 to D7 during T3 of PCI and PCR cycles, and of PCC input cycles | Instruction bytes and read data. The FPGA is the memory for the real chip |

Should the FPGA generate the clocks? Yes. It removes all clock-domain
questions: the phase counter that makes phi1 and phi2 also decides when the
soft core advances and when the real pins are sampled. One schedule that meets
the AC table with a 20 MHz tick (50 ns) [CALC]:

| Segment | Ticks | Time | Requirement |
|---|---|---|---|
| phi1 high | 15 | 750 ns | at least 700 ns |
| gap, phi1 fall to phi2 rise | 5 | 250 ns | tD3 at least 200 ns |
| phi2 high | 14 | 700 ns | at least 550 ns |
| gap, phi2 fall to phi1 rise | 10 | 500 ns | tD2 at least 400 ns |
| Period | 44 | 2.2 us | 2 to 3 us |
| phi1 fall to phi2 fall | 19 | 950 ns | tD1 0.90 to 1.1 us |

A state is two periods, so 88 ticks. Call the first period "a" (phi11, phi21)
and the second "b" (phi12, phi22), tick 0 at the rising edge of phi1.

Where to sample [CALC from p.14 and p.15]:

- Data out (T1, T2, T3 of a write) is valid at most 1.0 us after the falling
  edge of phi11, which is tick 35 of period a, and is held at least 0.1 us past
  the falling edge of phi22, which is tick 34 of period b. Sample late in
  period b, for example tick 28 to 30, while phi22 is still high.
- S0 to S2 are valid at most 1.1 us after the phi22 falling edge that ended
  the previous state, which is tick 12 of period a. The manual gives no hold
  time for them. Sample them together with the data at tick 28 to 30 of
  period b. They are certainly stable there.
- SYNC rises at most 0.7 us after phi22 falls and falls at most 0.7 us after
  phi21 falls. Sampled at tick 28 to 30 of every period it reads high in
  period a and low in period b.
- All of these pass through a clamp and a buffer and then need a two-flop
  synchroniser in the FPGA even though the FPGA makes the clock, because the
  8008's output delay is large and unknown.

Phase alignment: the FPGA decides when phi1 pulses happen but the 8008 decides
which pulse is phi11. After power-up, watch SYNC once to learn the parity, then
hold the soft core until the real chip is in a known state.

Getting both cores to the same starting state [DS 8008UM p.11]:

1. When VDD rises, an internal flip-flop forces a HALT and the chip enters
   STOPPED.
2. Sixteen clock periods in STOPPED with INT low clear the accumulator, the
   scratch pad, the program counter and the stack.
3. Raise INT on a rising edge of phi1, never near a falling edge of phi1 (see
   1.8 for what happens otherwise). The chip leaves STOPPED through T1I
   (S0 S1 S2 = 011). Drop INT after phi11 of the T1I state, not earlier.
4. In the T3 of that cycle, jam the first instruction. A one-byte RST is the
   usual choice. In a T1I cycle the program counter is not advanced.

The soft core should be given the same treatment: start it STOPPED, feed it
the same INT, the same jammed byte and the same memory contents. The manual
does not say the flags are cleared at power-up, so run a few
flag-setting instructions before any conditional branch or expect a possible
early mismatch [INFER].

Driving the bus in T3:

- The timing diagram shows the 8008 releasing the bus tOH after phi22 of T2,
  the external data window beginning at or after the falling edge of phi11 of
  T3 (marked "greater than or equal to 0"), and a hold of tIH after the falling
  edge of phi21 [DS 8008UM p.15]. That is my reading of a small diagram and
  should be confirmed on a scope.
- The state lines for T3 are only guaranteed valid 1.1 us after phi22 of T2
  falls, which is tick 12 of T3 period a, just before phi11 falls at tick 15.
  That is too late to use as the only enable. Predict T3 instead: the FPGA
  knows T2 is in progress and controls READY, so it knows T3 is next. Use the
  observed state lines to verify, not to decide.
- Intel did the same on its own board: "the T3 state is anticipated by setting
  a D-type flip-flop (A16) at the end of each T2 state. This line controls the
  multiplexing of data to the 8008. This flip-flop is reset at the end of each
  T3 state." [DS 8008UM PDF p.23]. The RetroShield firmware does it too
  (see 1.7).
- Anticipation is margin, not a necessity. Scott Baker's boards gate read data
  with the observed "T3 and SYNC" and run at 500 kHz (see 1.8).
- Enable the driver between the end of T2 and the middle of phi11 of T3, hold
  through phi21 and until SYNC has gone low, then release. The firmware enables
  about 400 ns after phi22 of T2 falls and tolerates the overlap. Do not drive
  during T1, T2, or T3 of a write cycle.
- Do not drive during T4 or T5. The SIM8-01 text says the 8008 normally has no
  interaction with the system in those states, but that for an INP instruction
  it sends its flag flip-flops out at T4 [Rev 4 manual in the firmware repo,
  OCR text, PDF p.33. The same paragraph starts at the bottom of 8008UM PDF
  p.23 in this repo].
- The same passage says switched pull-up resistors are used on the data bus
  "to minimize data bus loading and increase bus response" and that they are
  mandatory with the 8008-1 [DS 8008UM PDF p.23]. With one buffer per line
  the bus capacitance on a breakout is small, but keep this in mind if the
  8008-1 is run near 800 kHz.
- With a series resistor in each data line, a short overlap at either end is
  harmless.

What to compare: at each sample point compare the real S0 to S2 and SYNC
against the soft core's, and compare D0 to D7 whenever the cycle says the CPU
is driving (T1, T2, T3 of PCW, and T1/T2 of PCC). Internal registers of the
real chip are only visible through what it puts on the bus.

### 1.7 What the RetroShield firmware actually does [FW]

Source: `../retroshield-teensy/t8008/`. These sketches target the Teensy 4.1
only. The Teensy is a 3.3 V part and reaches the shield through the adapter in
`hardware/teensy`, which carries four TXS0108 auto-direction level translators
between 3.3 V and the 5 V header [SCH, part list only]. So the existing 3.3 V
solution is "translate to 5 V, then use the same 74LS245s". The 8008 still
sees LS245 levels.

Clock generation:

| Variant | phi1 high | phi1 fall to phi2 rise | phi2 high | phi2 fall to phi1 rise | Period | tD1 |
|---|---|---|---|---|---|---|
| `t8008_intellec_8`, fixed NOP delays | 700 ns | 400 ns | 650 ns | 400 ns plus code | about 2.15 us or more | 1050 ns |
| `t8008_scelbal_faster`, 8008 setting | 715 ns | 365 ns | 565 ns | 415 ns | about 2.06 us | 930 ns |
| `t8008_scelbal_faster`, 8008-1 setting | 365 ns | 215 ns | 365 ns | 365 ns | about 1.31 us | 580 ns |

These are the numbers written in the source. In the faster variant
`setuphold.h` adds a `BUFDELAY` of 15 ns to each segment (its comment says
50 ns) and times edges from the CPU cycle counter. I did not measure anything. All three follow the AC
table in 1.4, not the SIM8-01 figure, and the faster variant has compile-time
checks that tD1 stays between 900 and 1100 ns. The fixed-delay variant
stretches the last gap by however long the memory and serial code takes.

Start-up, as coded in `setup()`:

1. Clocks low, READY high, INT low.
2. The sketch waits for a USB serial connection before it does anything else.
   The 8008 therefore sits powered with both clocks stopped for an arbitrary
   time, seconds to minutes, and still starts correctly afterwards.
3. Clock until SYNC is seen high.
4. Run 20 states (40 clock periods) to let the chip clear itself.
5. Raise INT. In the Intellec variant INT is written at the end of the phi11
   high time, immediately before phi1 is driven low. That is inside the 200 ns
   window the manual says to avoid [DS 8008UM p.15]. The sketch evidently
   starts the chip, but a second project documented intermittent failures from
   exactly this coincidence (see 1.8), so treat it as a latent bug and do not
   copy it.
6. When the state lines read T1I, drop INT. In the following T3 drive the jam
   byte, by default 0x05 (RST 0).
7. The reset button does the same thing at run time: it just raises INT.

Per-state sequence in `cpu_tick()` of the Intellec variant:

| Point in the state | Action |
|---|---|
| End of phi11 high | Release the bus unless the previous state was T2 |
| End of phi21 high (650 ns into it) | Read S0 to S2. If T1I, drop INT |
| After phi21 falls, 400 ns later | Wait until SYNC reads low, then release the bus. Source comment: "DATA_IN hold time >= SYNC OUT delay" |
| After phi12 falls, 400 ns later | In T1 or T2, read the data bus (address low, or address high plus cycle bits) |
| End of phi22 high | In T3, read the data bus. Act on it if the cycle is PCW or PCC |
| After phi22 falls, 400 ns later | If this state was T2, the next will be T3: for PCI, PCR, a PCC input, or an interrupt cycle, switch the 74LS245 toward the 8008 and put the byte on the bus now |

What this confirms or adds:

- Direction control is predictive. The source says the state lines arrive too
  late to decide ("Chicken and egg"), so the bus is driven "as soon as we know
  next state will be T3", at the end of T2.
- The author expects contention there: "There will be a slight bus contention
  as 8008 changes its output and we start enabling outputs. 680 Ohm series
  resistors should prevent damage." A TODO asks whether 680 ohms should be
  reduced "to improve timing".
- The bus is released when SYNC falls in T3, which matches the tIH rule.
- State lines are sampled at the end of phi21 and data out during the second
  clock period. Both sit inside the windows worked out in 1.6.
- A source comment notes that the state lines take about 500 ns longer to
  settle in T1 and T1I.
- READY is held high. WAIT is not handled.
- Cycle type is decoded as bits 7 and 6 of the T2 byte: 00 PCI, 10 PCR, 01 PCC,
  11 PCW. That agrees with the manual's D6 D7 table once the bit order is
  accounted for.
- In a PCC cycle the T1 byte is the output data (register A) and bits 5 to 1
  of the T2 byte are the port number. The Intellec sketch drives T3 only when
  that number is 0 to 0x0B, which it treats as readable ports.
- `portmap.h` has an `INVERTED_CLK` option with the comment "Rev A board
  inverts clock via U2", and `cpu_tick()` comments mention data passing through
  "TXB0108 & 74HC245" and waiting for a "level shifter" to stabilise. So an
  earlier board revision existed with different buffering. It is not in either
  repo.

### 1.8 Second prior art: Scott Baker's 8008 "MasterBlaster" [SB]

Source: `../8008-super` (github.com/sbelectronics/8008-super), an eight-CPU 8008
machine built from one Master board and up to eight Blaster boards. What was
read: README, `NOTES.md`, Blaster schematic 0.14 pages 1 and 2 visually and
pages 3 to 5 as text, Master schematic 0.12 as text only, the Blaster bus PLD
source, and two of the sixteen captures in `scope/`.

Power [SB, Blaster schematic p.2. The Master sheet has the same text]:

- +8 V in through a barrel jack and fuse F4 to a 7805T, giving +5 V. 100 uF and
  10 uF on the input, 10 uF and 1 uF on the output. D4, a zener marked "5V",
  sits from +5 V to GND.
- -9 V comes from DC1, an isolated DC/DC module fed from +5 V. Its +VOUT is
  tied to GND and its -VOUT is the -9 V rail. The COM pin is not connected.
  10 uF on the input and 10 uF on the output. There is no linear regulator.
- D1, a zener marked "9V", and C2 1 uF sit from the -9 V rail to GND. A note on
  the sheet reads "Consider replacing C2 with a 1K resistor. It will ensure a
  minimum load on the converter."
- The schematic does not give a part number for DC1 or the zeners.

Compared with the RetroShield supply [INFER]: fewer parts and more current
headroom (a 1 W, 9 V module is about 110 mA [CALC]), but Vdd is only as
accurate as an unregulated module held down by a zener. The 8008 wants -9 V
within 5 %. The RetroShield's 15 V module plus 78L09 holds the rail much more
tightly. The minimum-load note is a real point for both designs: these
unregulated modules rise above nominal when lightly loaded.

Interface [SB, Blaster schematic p.1]. Everything is 74HCT or a GAL at 5 V,
wired straight to the 8008:

| Signal | Topology |
|---|---|
| D0 to D7 | Direct to the A side of a 74HCT245N. No series resistors. A 10k pull-up to +5 V on every line (RN6). DIR from the bus PLD (`DBD`) |
| S0, S1, S2, SYNC | Direct into ATF22V10 inputs. S0 also clocks a 74HCT74. No resistors, no clamps |
| phi1, phi2 | 74HCT02 NOR gate outputs, direct, each with a 1k pull-up to +5 V (RN7) |
| READY | 1k pull-up to +5 V |
| INT | From the I/O PLD, 1k pull-up to +5 V |
| Supply bypass | A capacitor directly between +5 V and -9 V at the chip, added in rev 0.14 ("Add missing bypass cap on 8008") |

Clock: a 1 MHz can oscillator, a 74HCT74 dividing by two, and two NOR gates.
That gives phi1 and phi2 each 500 ns wide with 500 ns gaps and a 2 us period
[INFER from the logic. The captures at 2 us per division agree]. This is the
SIM8-01 figure, not the AC table, which wants phi1 at least 700 ns wide.
`NOTES.md` calls it "the noncompliant clock design in the Bayles and Loos
projects, which I inherited". Eight CPUs run on it. The author tried both a
D8008 and a D8008-1 and "experienced no difference".

Bus control, from `pld/blaster-bus/bus.pld` ("Original by Jim Loos"):

```
DBD    = (t1 # t1i # t2 # t3) & !SYNC;
ALL    = (t1 # t1i) & !SYNC & PHASE2;
ALH    = t2 & !SYNC & PHASE2;
MEMRD  = t3 & (pci # pcr) & SYNC;
MEMWR  = t3 & pcw & !SYNC & PHASE2;
IORD   = t3 & pcc & SYNC & (!LOSTIT);
IOWR   = t3 & pcc & !SYNC & PHASE2;
INTACK = t1i & !SYNC & PHASE2;
```

- Address bytes are latched during phi22 of T1 and T2, which is the sample
  point recommended in 1.6.
- Read data is put on the bus while the observed state is T3 and SYNC is high,
  and removed when SYNC falls. There is no anticipation of T3 here.
- The state and cycle decodes match the manual and the RetroShield firmware.
- A comment mentions an unresolved "stability issue" around `DBD`, and the
  Master PLD notes that I/O strobes must be qualified with the port decode or
  spurious pulses appear.

The interrupt problem. This is the most useful thing in the repo. From
`NOTES.md`: waking a halted CPU with INT usually gave T1I as expected, but some
captures showed the CPU entering T1 instead, running an instruction, and only
then acknowledging, and one showed a sliver of T1I inside a T1 followed by an
unexplained WAIT. The cause: "a coincidence between the O1 clock and the INT
line. If INT goes up at approximately the time O1 goes down, then bad things
will happen." The fix was a flip-flop that passes INT to the CPU only on the
rising edge of phi1 (rev 0.13. Rev 0.12 used the rising edge of phi2). The
same notes say that dropping INT as soon as the CPU starts running leaves it
halted, with the suspicion that "the CPU is sampling INT at the start of the
T1 cycle". The capture `blaster-cpuint-on-clk1-annotated.png` shows the bad
case: STOPPED, then T1 and T2 with no T1I.

Other points from the notes: after reset the Blaster "expends 16 clock periods
resetting itself" and stops. Waking from HALT runs the next instruction twice
because T1I does not advance the program counter, so a NOP follows every HALT.

What this means for the FPGA board:

1. It is an independent, working example of the approach recommended in 3.8:
   74HCT at 5 V on every 8008 pin, plus the pull-ups Intel asks for.
2. It runs with 8008 outputs wired straight into HCT and GAL inputs, so those
   inputs' protection diodes cope with the negative swing in practice [INFER].
   That is not a guarantee, and a 3.3 V LVC input is a different part. Keep
   the series resistor and Schottky.
3. By its `DBD` equation the 74HCT245 drives the 8008 data pins during the
   first half of T1 and T2 while the 8008 is turning its own outputs on, with
   no series resistance [INFER]. So direct contention is survivable. Keep the
   resistors anyway.
4. Gating read data with "T3 and SYNC" works at 500 kHz without anticipating
   T3. The window is evidently less tight than the worst-case numbers in 1.6
   suggest. Anticipation, as Intel and the RetroShield do it, buys margin.
5. INT timing is a hard requirement, not a datasheet nicety. Change INT only on
   the rising edge of phi1 and hold it until T1I has been seen. The RetroShield
   Intellec sketch raises INT just before phi1 falls (1.7). That is exactly
   the failing case here, so do not copy it.
6. A 500 ns phi1 works on real chips. For a lockstep board stay with the AC
   table anyway.
7. The captures are logic analyzer traces. They say nothing about voltage
   levels, so the question of how far outputs swing negative is still open.

---

## 2. Intel 4004: RetroShield k4004s

### 2.1 What is in the repo

`hardware/k4004s/` is a gEDA project: `k4004s.sch` (schematic source),
`k4004s.pcb` (layout source), PDFs and PNGs of both, a `symbols/` folder, and a
makefile. No BOM and no README.

The netlist below is exact. It was produced by converting `k4004s.pcb` to KiCad
with `kicad-cli pcb import` and reading pad-to-net assignments from the result
(43 footprints, 52 nets). Component values come from the `.sch` attributes. See
Appendix C.

### 2.2 Power rails [SCH]

This board does not use a negative rail. It shifts the 4004 up instead:

| 4004 pin | RetroShield connection |
|---|---|
| VSS (pin 5) | Net `P15V`, +15 V |
| VDD (pin 12) | GND, 0 V |

The datasheet only requires VSS - VDD = 15 V +/-5 % [DS 4004 p.7], so +15 V
and 0 V is as valid as +5 V and -10 V.

How +15 V is made:

- U9 is the same RECOM ROE-0515S module as on the 8008 board, 5 V in, 15 V at
  66 mA out, unregulated [DS ROE0515S.pdf p.1]. Here -Vout is tied to system
  GND, so the isolation is not used.
- C4 4.7 uF on the output, R10 10k to GND marked "Dummy load for regulation".
- J2 is a 3-pin jumper: pin 1 is the 15 V output, pin 2 is `P15V`, pin 3 is
  +5 V. The schematic labels it "15V or 5V option". What the 5 V position is
  for is not documented in the repo.
- Schematic note: "4004 requires 40mA max", which matches IDD 30 mA typical,
  40 mA maximum [DS 4004 p.7].
- Output accuracy is +/-5 % and load regulation is 10 % from 20 % to 100 %
  load [DS ROE0515S.pdf]. With the 4004 drawing 30 to 40 mA the module is at
  about half load. With the socket empty the only load is the 10k resistor
  (1.5 mA), and an unregulated module will then sit well above 15 V [INFER].
- Decoupling: C6, C7, C8 100 nF and C3 4.7 uF on +5 V. C9, C10, C11 100 nF
  (25 V) on `P15V`.
- LED "P15V": `P15V` to R20 3.1k to LED to D1 5.6 V zener to GND. Schematic
  note: "LED turns on at 15V only".

Sequencing: none. `docs/k4004/k4004_notes.rtf` mentions an `EN_BOOST_15V`
output, but that signal does not exist in this schematic revision.

### 2.3 Per-signal table [SCH]

"Dir" is relative to the 4004. All paths are non-inverting.

| Signal | 4004 pin | Dir | Topology | Parts and values | MCU pin |
|---|---|---|---|---|---|
| D0 to D3, toward CPU | 1 to 4 | bidir | MCU to U2 input. U2 output (0 to 15 V, tri-state) to 680 ohm series resistor to bus | U2 CD40109B (VCC = +5 V, VDD = P15V), R6 to R9 680 ohm | PC0 to PC3 (D37 to D34) |
| D0 to D3, from CPU | 1 to 4 | bidir | Bus direct to U3 input. U3 output (0 to 5 V) to MCU | U3 CD4504B used as a down-shifter (VCC pin = P15V, VDD pin = +5 V, SELECT = GND) | PL0 to PL3 (D49 to D46) |
| D0 to D3 pull-downs | 1 to 4 | n/a | 10k from each bus line to GND, which is the 4004's VDD | R35, R34, R33, R32 10k | |
| Bus direction | n/a | n/a | MCU GPIO to all four U2 enables. High means U2 drives the bus. 10k pull-down so the default is released | net `DRIVE_DBUS`, R3 10k | PD7 (D38) |
| SYNC | 8 | out | Pin direct to U5 input, 10k pull-down to GND. U5 output to MCU | U5 CD4504B down-shifter (same wiring as U3), R31 10k | PA5 (D27) |
| CM-ROM | 11 | out | Same | U5, R30 10k | PA4 (D26) |
| CM-RAM0 to CM-RAM3 | 16, 15, 14, 13 | out | Same | U5, R29, R28, R27, R26 10k | PA0 to PA3 (D22 to D25) |
| phi1 | 6 | in | MCU to U4 input, U4 output (0 to 15 V) direct to pin | U4 CD4504B up-shifter (VCC = +5 V, VDD = P15V, SELECT = GND). No resistor | PG1 (D40) |
| phi2 | 7 | in | Same | U4 | PG0 (D41) |
| RESET | 9 | in | Same | U4 | PB1 (D52) |
| TEST | 10 | in | Same | U4 | PB3 (D50) |

Details:

- CD4504B SELECT (pin 13) is grounded on U3, U4 and U5. SELECT low is
  CMOS-to-CMOS mode. SELECT high would be TTL input mode [DS cd4504b.pdf p.1].
- Unused CD4504B inputs are grounded (two on U3, two on U4).
- The schematic note beside the ten pull-downs reads "Needed for 4004 to drive
  high (VDD)". Read "high" as logic 1: the 4004 uses negative logic, so logic 1
  is the level near VDD [DS 4004 p.7].
- `k4004_notes.rtf`: "Remember to invert bits for 4004. Arduino 1 means 0 to
  4004 and vice versa."
- Four LEDs Q0 to Q3 with 680 ohm resistors hang on MCU pins PC4 to PC7 to
  show emulated RAM output port bits. They do not touch the 4004.

### 2.4 What the 4004 datasheet requires

DC [DS 4004 p.7], with the same limits expressed in both rail conventions
[CALC]:

| Parameter | Limit | VSS = +5, VDD = -10 | VSS = +15, VDD = 0 |
|---|---|---|---|
| VIH, except clocks | VSS - 1.5 min, VSS + 0.3 max | 3.5 V min | 13.5 V min |
| VIL, except clocks | VDD min, VSS - 5.5 max | -0.5 V max | 9.5 V max |
| VIL, TEST input | VSS - 4.2 max | 0.8 V max | 10.8 V max |
| VIH, clocks | VSS - 1.5 min | 3.5 V min | 13.5 V min |
| VIL, clocks | VDD min, VSS - 13.4 max | -8.4 V max | 1.6 V max |
| VOH | VSS - 0.5 min, capacitive load | 4.5 V min | 14.5 V min |
| VOL, data bus, CM, SYNC | VSS - 12 min, VSS - 6.5 max at 0.5 mA | -7 to -1.5 V | 3 to 8.5 V |
| Output resistance at the VSS level | data 150 typ, 250 max ohm. CM-ROM 320, 600. CM-RAM 1.1k, 1.8k | | |
| IDD | 30 mA typ, 40 mA max | | |
| Absolute maximum, inputs and supply relative to VSS | +0.5 V to -20 V | | |

Capacitance: clock inputs 14 pF typical, 20 pF maximum. Data bus 7 and 10 pF.
Other inputs and outputs 10 pF maximum [DS 4004 p.7].

AC [DS 4004 p.8]:

| Parameter | Min | Max |
|---|---|---|
| Clock period | 1.35 us | 2.0 us |
| Clock rise and fall | | 50 ns |
| Clock width | 380 ns | 480 ns |
| Delay phi1 to phi2 | 400 ns | 550 ns |
| Delay phi2 to phi1 | 150 ns | |
| Data-in, CM, SYNC write (setup) time | 350 ns | |
| Data-in, CM, SYNC hold time | 40 ns | |
| Data bus hold during M2 to X1 and X2 to X3 | 150 ns | |
| Data out access, data lines | | 930 ns at 500 pF, 700 ns at 200 pF |
| Data out access, SYNC, CM-ROM, CM-RAM | | 930 ns |
| Data out hold | 50 ns | |

Details that only became legible in the cleaner scan [DS 4004 p.8 and p.9]:

- Data in, CM and SYNC inputs are referenced to the phi2 pulse. They must be
  valid tW (350 ns) before the trailing edge of phi2 and stay valid tH (40 ns)
  after it. With phi2 only 380 to 480 ns wide, that means valid from about the
  leading edge of phi2.
- Outputs are clocked out by the trailing edge of phi2. The access times are
  measured from that edge (note 2).
- The timing detail draws the input and output thresholds at 1 V and 5 V below
  VSS.
- Note 3: every MCS-40 part that sends data to the 4004 at M2 and X2 floats
  afterwards, and the 150 ns bus hold requirement "is always insured since
  each component contributes 10 uA of leakage current and 10 pF of capacitance
  which guarantees that the data bus cannot change faster than 1 V/us".

Note 3 matters for any board with resistor pull-downs on the data bus. Intel's
hold time relies on a floating bus keeping its charge. A 10k pull-down into
about 40 pF moves the bus at roughly 37 V/us to begin with, and a line left at
VSS falls below VSS - 1.5 V in about 40 ns [CALC]. So with pull-downs fitted
the external driver must keep driving for the 150 ns after M2 and after X2
and only then release. The RetroShield does this in effect: it releases at the
start of X1, at least 150 ns after phi2 ends, and the CD40109B then takes
300 ns or more to let go [FW, DS cd40109b.pdf p.4]. It also explains why
Intel's SIM4 board switched its pull-downs in only during M1 and M2
[DS MCS-4_UM PDF p.61].

The clocks rest high (at VSS) and pulse low. An instruction cycle is eight
clock periods: A1, A2, A3 (CPU drives the address), M1, M2 (ROM drives, CPU
reads), X1 (CPU), X2 (CPU, or an I/O or RAM device on a read), X3 (CPU). SYNC
marks X3 [DS 4004 p.9 Figure 1].

Intel's own recommendation for mixing MCS-4 parts with TTL
[DS MCS-4_UM PDF p.101, Figure 38, and PDF p.106 note 1]:

- Use VSS = +5 V +/-5 % and VDD = -10 V +/-5 %.
- "For TTL compatibility, a resistor should be added between the output and
  VDD. All outputs are push-pull MOS outputs." The figure shows 12k for a
  4001/4002 output port and 5.6k for a 4003 output.
- The SIM4 board description mentions discrete TTL-to-MOS level shifters on
  the data bus whose pull-down resistors are connected "via diode disconnects"
  and only activated during M1 and M2 [DS MCS-4_UM PDF p.61]. I did not find
  or read the schematic of that circuit.

### 2.5 How the RetroShield meets, or does not meet, those numbers

Driving the 4004:

- CD4504B and CD40109B outputs swing rail to rail (0.05 V and VDD - 0.05 V
  unloaded) [DS cd4504b.pdf p.2, cd40109b.pdf p.3]. Their VDD pin is the same
  `P15V` net as the 4004's VSS, so an input can never be driven above VSS and
  the high level tracks the supply.
- Clocks, RESET and TEST get the full 0 to 15 V swing with no resistor. The
  clock low limit of 1.6 V is met easily.
- Clock edges: CD4504B output transition time at VDD = 15 V and 50 pF is 40 ns
  typical, 80 ns maximum [DS cd4504b.pdf p.3]. The 4004 allows 50 ns. Typical
  parts into about 20 to 30 pF will pass. The datasheet maximum does not.
- Data bus high level: U2 drives through 680 ohms into the 10k pull-down, so
  the pin sees 15 x 10k / 10.68k = 14.04 V [CALC]. The limit is 13.5 V. The
  margin is about 0.5 V.
- U4 is in CMOS mode with VCC = 5 V, so it needs 3.5 V for a high
  [DS cd4504b.pdf p.2]. A 5 V Arduino meets that. A 3.3 V FPGA would not.

Receiving from the 4004:

- High level 14.5 V into U3 and U5, whose input thresholds scale with their
  15 V VCC pin. Fine.
- Low level: the 4004 only guarantees 8.5 V or lower at 0.5 mA. A CD4504B in
  CMOS mode with VCC = 15 V wants roughly 4.5 V or lower (30 % of VCC,
  extrapolated from the 5 V and 10 V rows of the table [DS cd4504b.pdf p.2]).
  The 10k pull-downs to the 4004's VDD are what finish the swing, which is
  exactly what the schematic note says and what Intel's Figure 38 does.
- Down-shifting delay through a CD4504B from 15 V to 5 V is 275 ns typical,
  550 ns maximum falling and 200 ns typical, 400 ns maximum rising
  [DS cd4504b.pdf p.3].

Bus turnaround:

- Direction is a firmware decision (`DRIVE_DBUS`), made by counting clock
  periods from SYNC. `k4004_notes.rtf` lists `DRIVE_DBUS_START` and
  `DRIVE_DBUS_STOP` macros.
- CD40109B enable and disable times at VCC = 5 V, VDD = 15 V are slow: 230 to
  460 ns to enable high, 300 to 600 ns to release from low
  [DS cd40109b.pdf p.4]. Against a 1.35 to 2.0 us clock period that leaves
  little room. My first guess was that the board must run the 4004 slowly.
  The Teensy firmware shows otherwise [FW, `t4004_empty`]: it sets 740 kHz on
  a Teensy 4.1 with phase widths of 380, 400 and 380 ns taken from the
  datasheet (500 kHz with stretched phases on a Teensy 3.5), and after
  writing the bus it loops until
  the value read back through the down-shifter matches what it wrote
  ("Wait until level shifter output stabilizes"). So it runs at the fast end
  of the specification and closes the loop on the shifter delay instead of
  assuming it.
- The firmware inverts every data and status bit in its pin macros, which is
  the negative-logic conversion.
- The original Arduino Mega sketch is the opposite case
  [FW, `retroshield-arduino/k4004/archive/k4004_busicom`]. It has no timed
  delays at all: `LEVEL_SHIFT_PROP_DELAY` is defined empty with the comment
  "inherent delay from ATMEL instructions", each clock phase is two
  back-to-back port writes, and the direct port macros are described as
  "required to meet >100kHz clock". It also does keyboard, printer and serial
  work between instruction cycles. So on a Mega the 4004 is clocked well below
  its 500 kHz minimum rate (2.0 us maximum period) and with an uneven period,
  and the Busicom calculator ROM still runs. No rate was measured by me.
- That sketch samples the bus while phi1 is low. It starts driving at M1 while
  phi1 is low and releases at the start of X1. For I/O and RAM reads it drives
  again in X2 and releases in X3. It also releases at the start of A1.
- The later `kt4004_busicom_v2` sketch adds explicit delays of about 360 ns
  per phase, with the comment "550ns max prop delay in CD4504B", which is the
  datasheet figure quoted above.
- The 680 ohm resistors limit contention current. 15 V across 680 ohms is
  22 mA [CALC].

### 2.6 The benefactor's suggested 4004 interface, checked

the chip donor sent two circuit ideas on 2026-11-06, both assuming
VSS = +15 V and VDD = ground. Tag for this whole section: the donor's text is
[MEM]-grade input from an experienced engineer, and every number below that
checks it is [DS] or [CALC] unless marked otherwise.

His proposal:

1. 4004 outputs into an LM2901 quad comparator running at VCC = 5 V, with a
   resistive divider making a +3.5 V reference, giving TTL out directly.
2. 4004 inputs from an open-drain N-channel MOSFET (BS170, "2.1 V switching
   threshold"), source grounded, drain pulled up to +15 V. Pull-up weak enough
   that the 4004 can drive against it on the bus.

What checks out:

- The comparator idea is the only receive scheme seen so far that meets the
  datasheet with guaranteed margin. The 4004 promises a low of only VSS - 6.5 V
  at 0.5 mA, which is 8.5 V above ground in the +15 V arrangement [DS 4004
  p.7]. The RetroShield's CD4504B wants about 4.5 V or lower for a low, so it
  works on typical parts only (section 2.5). A comparator with its threshold
  placed between 8.5 V and 14.5 V reads any in-spec 4004.
- The open-drain FET is the right shape for the clocks, TEST and RESET. A
  clock low must be within 1.6 V of ground and the FET gets close to 0 V. The
  high is the pull-up resistor, and nothing else, so the 50 ns rise limit sets
  the resistor: with 20 pF of clock input plus about 10 pF of strays, 10 to
  90 % takes about 2.2 RC, so R must be 750 ohms or less [CALC]. 680 ohms
  gives 22 mA through the FET while the clock is low. The clocks are low about
  a third of the time each (380 to 480 ns of 1350 ns), so the average is about
  15 mA for both clocks on top of IDD at 30 to 40 mA [CALC]. That is 45 to
  55 mA against the ROE-0515S limit of 66 mA. Tight. The 2 W TMH 0515S in the
  experiment order has room.

What does not check out, or needs a datasheet I do not have:

- The 3.5 V threshold does not fit +15 V and ground. A low can legitimately
  sit anywhere from 3 V to 8.5 V above ground, so a 3.5 V reference misreads
  most in-spec lows as highs. The threshold belongs near 11.5 V, midway
  between the 8.5 V worst-case low and the 14.5 V worst-case high. A 3.5 V
  reference does fit the +5 V and -10 V arrangement (high 4.5 V or above, low
  -1.5 V or below), so I suspect the two conventions got mixed in the note.
- An LM2901 at VCC = 5 V has an input common-mode range of 0 V to VCC - 1.5 V
  [MEM]. A 14.5 V input is far outside it. The family is rated to take inputs
  up to 36 V without damage [MEM], but whether the output is still correct
  with one input that far above the range is a datasheet question and the
  LM2901 datasheet is not in the repo. (Superseded by 2.7: the datasheet
  says it is correct as long as the reference input stays in range.) The clean version of the same idea: run
  the comparator from the +15 V rail, divide +15 V down to about 11.5 V for
  the reference, and pull the open-collector output up to 3.3 V. That gives
  an FPGA-level signal with no further shifting at all.
- Speed. From memory the LM2901 response time is about 1.3 us with 5 mV of
  overdrive and a few hundred nanoseconds with volts of overdrive [MEM,
  unverified]. Against a 350 ns setup window and a 930 ns access time that is
  a large fraction of the budget, and on a lockstep board it lands straight in
  the sampling margin. Faster open-collector comparators that still take 15 V
  on the inputs exist (LM319 is the one I would check first, two per DIP-14).
  Needs a datasheet pass before committing.
- "Weak enough that the 4004 can drive against it" is the right constraint on
  the bus pull-up but it has a cost the note did not mention. The 4004 sinks only
  0.5 mA at its guaranteed low, so a pull-up to +15 V must be about 13k or
  more [CALC: (15 - 8.5) / 0.5 mA]. When the FET releases, the bus then rises
  through 13k into about 30 pF, a 0.4 us time constant, and takes about 1 us
  to reach the 13.5 V input high level [CALC]. The 4004 wants data valid 350 ns
  before the end of phi2. So the bus lines need a switched pull-up (what
  Intel's SIM4 did with its diode disconnects) or a tri-state push-pull driver
  (what the CD40109B provides). Clocks, TEST and RESET are not driven by the
  4004 and can use the strong pull-up.
- BS170 threshold. The 2.1 V is the typical figure; the datasheet range is
  0.8 V to 3.0 V [MEM], and the on resistance is specified at VGS = 10 V. A
  3.3 V FPGA pin cannot be relied on to turn one on hard enough to sink 22 mA
  with a clean low. Drive the gates from the 5 V 74AHCT541 stage that the
  8008 board needs anyway. Same caution applies to any TO-92 FET; 2N7000 is
  the same class.

What this means for the +5 V and -10 V decision in section 3.2:

- The drive side turns out to be the same circuit mirrored. With VSS at +5 V
  the FET becomes a P-channel part (BS250 class) with its source at +5 V and
  its gate driven from 5 V logic, and the resistor pulls the pin down to
  -10 V. The high is the FET, the low is the resistor. Same 680 ohm limit for
  the clocks, same 22 mA, except that the current flows while the clock rests
  high, about two thirds of the time, so the average is about 31 mA rather
  than 15 mA [CALC]. Still inside the 2 W module. Same bus problem: a
  resistor the 4004 can drive high against (ROH 250 ohms max, so 1.3k or
  more for a 2.5 V high at a TTL receiver; Intel's 12k gives 4.7 V) is too
  slow to make the low, so the bus needs a push-pull tri-state driver in this
  arrangement too [CALC].
- The receive side is where the two arrangements differ. At +15 V and ground
  the low never gets near ground, so a comparator is needed. At +5 V and
  -10 V the worst-case low is -1.5 V, below ground, so a series resistor, a
  Schottky clamp to ground and a 74AHCT541 at 5 V give a TTL low with
  guaranteed margin and the 4.5 V minimum high clears the 2 V TTL threshold
  by 2.5 V. No reference, no comparator, no comparator delay, and it is the
  8008 receive circuit unchanged. With a 10k series resistor the clamp draws
  about 0.1 mA from a pin sitting at -1.5 V, inside the 0.5 mA budget [CALC].
  Intel's pull-down to VDD on each output stays.
- So on 2026-11-06 the decision stood on the evidence. the donor's two ideas are
  both sound for the donor's arrangement and the drive side is equivalent either way.
  The shifted-ground arrangement keeps the simpler receive side and the shared
  8008 circuit. The one thing the +15 V arrangement wins is that no
  regulator sits between the module and the chip.

Decision revised 2026-11-13: the chip actually being sent is a 4004, with no
8008. The shared receive circuit was the only argument for +5 V and -10 V, and
it no longer applies. The first 4004 board uses the donor's arrangement, VSS at
+15 V and VDD at ground, with the donor's two circuits:

- Inputs: BS170 open drain, source to ground, drain to the pin, pull-up to
  +15 V. 680 ohms on the clocks for the 50 ns edge, 10k on TEST and RESET.
  Gates driven from a 74AHCT541 at 5 V, not straight from the 3.3 V FPGA.
- Bus lines: BS170 for the low, a P-channel TO-92 FET from +15 V for the high
  with its gate driven by a CD4504B channel, both off for tri-state. One
  more open question closed (3.6 item 11) in principle, untested.
- Outputs: LM2901 comparators powered from +15 V, reference about 11.5 V from a
  divider, open-collector outputs pulled up to 3.3 V, straight to the FPGA.
  LM319 held in reserve if the LM2901 is too slow on the bus.
- Rail: a regulated isolated module (TRACO TMR 1-0513, single 15 V, 67 mA; the TMR 1-0523 first listed is a dual +/-15 V part at 33 mA per output, caught on the Mouser product page 2026-11-13) so the
  chip sees 15 V within 5 % without a bench supply, plus one ROE-0515S to
  measure how far the unregulated RetroShield rail drifts. Bench supply with
  a 60 mA current limit for the first power-up of the real chip.

### 2.7 Datasheet check of the parts in the donor's circuit

Read 2026-11-13. onsemi BS170 datasheet, BS170/D Rev. 7, November 2017 [DS].
TI LM139/LM239/LM339/LM2901 datasheet, SLCS006Z, revised May 2025 [DS].
Neither is in the repo; both were downloaded from the manufacturer.

BS170 [DS BS170/D p.2 table, p.3 Figures 3, 4, 6]:

| Parameter | Datasheet | Note |
|---|---|---|
| VGS(th) | 0.8 min, 2.0 typ, 3.0 max V at ID = 1 mA | the donor's "2.1 V" is the typical. Worst case 3.0 V |
| rDS(on) | 1.8 typ, 5.0 max ohm at VGS = 10 V, ID = 200 mA | Only guaranteed at 10 V gate drive |
| Figure 4, on region, typical | at VGS = 5 V about 0.3 A at VDS = 1 V, so roughly 3 ohms; at VGS = 4 V about 0.1 A, roughly 10 ohms | Typical curves, not limits. No curve below 4 V |
| ID continuous | 0.5 A | |
| V(BR)DSS | 60 V min | Fine for 15 V |
| Ciss, Coss | 60 pF max; Coss about 20 pF at VDS = 15 V from Figure 6 | Coss sits on the 4004 pin |
| ton, toff | 4 ns typ, 10 ns max at 0.2 A | Not the limit on clock edges |
| VGS(th) vs temperature | falls about 0.4 V from 25 C to 100 C | Helps, does not rescue a 3.3 V drive |

Conclusions for the drive side:

- Gate drive at 5 V from the 74AHCT541: a worst-case 3.0 V threshold part
  still has 2 V of overdrive, the typical curve shows about 3 ohms, and the
  22 mA clock pull-up would drop under 0.1 V. Good.
- Gate drive straight from a 3.3 V FPGA pin: a 3.0 V threshold part has 0.3 V
  of overdrive and the datasheet defines threshold at 1 mA. Not good enough
  for 22 mA. Keep the 5 V stage.
- Coss changes the clock pull-up arithmetic from section 2.6. The clock node
  is about 20 pF of 4004 input, 20 pF of BS170 Coss and 10 pF of strays, 50 pF
  in all. For a 50 ns 10 to 90 % rise, R must be 450 ohms or less, not 680
  [CALC: 50 ns / (2.2 x 50 pF)]. That is 33 mA per clock while low, and about
  22 mA average for both clocks, which with IDD at 40 mA sits at the 67 mA
  limit of the TMR 1-0513 [CALC]. Too tight.
- Fix: do not use a pull-up resistor on the clocks at all. Drive each clock
  from a CD4504B channel directly, which is what the RetroShield does, at
  40 ns typical and 80 ns maximum into 50 pF [DS cd4504b.pdf p.3]. One
  CD4504B has six channels: two clocks plus four P-FET gates for the bus
  driver. BS170 open drain stays for TEST and RESET, where a 10k pull-up and a
  slow edge are fine. No static pull-up current anywhere, and the regulated
  1 W module has headroom.
- Edge rate caveat stands: nothing here guarantees 50 ns on a worst-case
  CD4504B. Typical parts meet it. Measure once a scope is involved.

LM2901 [DS SLCS006Z sections 6.1, 6.9, 6.13, notes]:

| Parameter | Datasheet | Note |
|---|---|---|
| Supply | 2 to 36 V (30 V for the LM2901 electrical table) | 15 V is fine |
| Input voltage, absolute maximum | -0.3 V to 36 V, either input | Nothing negative in the +15 V arrangement |
| VICR | 0 to VCC - 1.5 V at 25 C, 0 to VCC - 2 V over temperature | |
| Note on VICR | "one input can exceed VCC, and the comparator provides a proper output state as long as the other input remains in the common-mode range. Either or both inputs can go to 30 V without damage." | the donor was right, section 2.6 was wrong on this point |
| Response time | 1.3 us typ, 100 mV step with 5 mV overdrive; 0.3 us typ, TTL-level input step. No maximum. | |
| Output | open collector; IOL 6 mA min at VOL 1.5 V; VOL 400 mV max at 4 mA | Pull-up to 3.3 V is allowed |
| Input bias current | 25 nA typ, 250 nA max | Invisible against the 4004's 10 uA leakage |

Conclusions for the receive side:

- My common-mode objection in 2.6 is withdrawn. A 4004 output at 14.5 V into
  an LM2901 running at VCC = 5 V is explicitly allowed and gives a correct
  output, provided the reference input is inside the range.
- The threshold objection stands and gets a reason. At VCC = 5 V the
  reference must stay below VCC - 2 V, so 3 V over temperature. That is why
  the donor's 3.5 V appears: it is the highest reference a 5 V comparator can
  take. But the 4004 low can be as high as 8.5 V, so a 3 V reference reads
  in-spec lows as highs. The threshold has to sit near 11.5 V, and a
  reference at 11.5 V needs VCC of at least 13.5 V. So the comparator runs
  from the +15 V rail. Reference 11.5 V is then 1.5 V inside VICR over
  temperature.
- Speed: with the threshold at 11.5 V the overdrive is at least 3 V either
  way, so expect the 0.3 us typical class of response, not 1.3 us. There is
  no maximum in the datasheet. For a blink test it does not matter. For bus
  reads it is 0.3 us of a 0.93 us access budget, which is why the LM319 is in
  the order as a fallback.
- Output: open collector pulled up to 3.3 V through 4.7k is 0.7 mA, far
  inside the 6 mA minimum sink, and VOL stays under 0.4 V. The FPGA input
  never sees anything above 3.3 V. No 74AHC541 stage needed on the receive
  side.

TP0606N3 P-channel, Supertex/Microchip datasheet, 4 pages, downloaded from a
university mirror because the Microchip and Mouser servers refused the fetch
[DS TP0606 p.1 and p.2 table]:

| Parameter | Datasheet | Note |
|---|---|---|
| BVDSS | -60 V | |
| VGS maximum | +/-20 V (IGSS test condition) | CD4504B gate drive gives -15 V on. Inside |
| VGS(th) | -1.0 min, -2.4 max V at ID = -1 mA | |
| RDS(on) | 5.0 typ, 7.0 max ohm at VGS = -5 V, ID = -250 mA; 3.0 typ, 3.5 max at -10 V | Guaranteed at a 5 V gate, which the BS170 is not |
| ID continuous | 0.32 A in TO-92 | |
| CISS | 80 typ, 150 max pF | Heavier gate than the BS170; the CD4504B channel drives it |
| tr, tf | 15 ns typ at 1 A | |

Conclusion: a good high-side switch for the bus driver. With its gate at the
CD4504B's 0 V output, VGS is -15 V and it is fully on; at 15 V it is off.
Through a 680 ohm series resistor to the bus (contention limit, same as the
RetroShield) the on resistance is invisible.

LM319, STMicroelectronics LM119/LM219/LM319 datasheet, downloaded from a
university mirror, then confirmed line by line against the TI LM119/LM219/
LM319 datasheet SNOSBJ2 (the part in the order). Same limits in both
[DS LM319 ST Tables 1 and 3; DS TI SNOSBJ2 sections 6.1, 6.5, 6.6]:

| Parameter | Datasheet | Note |
|---|---|---|
| Supply | 5 V single to +/-15 V; VCC+ absolute maximum 18 V | Single 15 V allowed |
| Differential input, absolute maximum | +/-5 V | The problem, see below |
| Input voltage, absolute maximum | +/-15 V, and "equal to the supply voltage" below +/-15 V supplies | 14.5 V at a 15 V supply is at the limit |
| Vicm | +/-12 V min at +/-15 V; 1 to 3 V at +5 V single | Roughly 3 V inside each rail. No single 15 V row |
| Response time | 80 ns typ, 100 mV step, 5 mV overdrive | |
| VOL | 0.4 V max at 3.2 mA sink with VCC- = 0 V | Open collector, 3.3 V pull-up fine |
| Input bias | 250 nA typ, 1 uA max (LM319) | |
| Supply current | about 8 mA typ per package at +/-15 V, 4.3 mA at 5 V | From the 15 V rail, two comparators per package |

Conclusions:

- The +/-5 V differential limit bites. With the reference at 11.5 V a 4004
  low at the datasheet's 3 V puts 8.5 V across the inputs. The LM2901 has no
  such limit (its inputs are rated independently to 36 V); the LM319 does.
  So the LM319 cannot take the 4004 output directly. The fix is a 2:1
  divider on the signal input with the reference halved to about 5.75 V:
  the high becomes 7.25 V, the worst lows 1.5 to 4.25 V, differentials
  +1.5 V and -4.25 V, inside +/-5 V, and the common mode stays inside the
  1 to 12 V window [CALC]. The divider costs nothing in speed at these
  impedances if it is 10k and 10k or lower.
- Common mode at the top: 11.5 V is 3.5 V below a 15 V supply, right at the
  edge of what the +/-15 V row implies. The divided version at 5.75 V has no
  such worry. Use the divider.
- Supply current is the other surprise. Five LM319 packages for nine outputs
  would draw about 40 mA from the 15 V rail, more than the 4004 itself, and
  blow the 67 mA budget. Three LM2901 packages draw about 2.4 mA typical,
  6 mA max. So the LM2901 stays the default for all nine outputs and the one
  LM319 in the order is for the four bus lines only, if the LM2901's 0.3 us
  turns out to matter. Two packages, about 16 mA.

TMR 1-0513, Traco TMR 1 series datasheet dated October 8, 2026, saved as
`datasheets/tmr1_datasheet.pdf` [DS TMR1 p.1 and p.2]:

| Parameter | Datasheet | Note |
|---|---|---|
| Model | TMR 1-0513: 4.5 to 9 V in, 15 V at 67 mA out, 79 % efficiency | Single output. The 0523 is +/-15 V |
| Voltage set accuracy | +/-1 % max | 14.85 to 15.15 V before regulation terms |
| Line regulation | 0.2 % max over 4.5 to 9 V in | |
| Load regulation | 0.5 % max, 10 to 90 % load | 75 mV. The ROE-0515S is unregulated |
| Temperature coefficient | +/-0.02 %/K max | |
| Worst case stack | about +/-1.7 % plus 0.3 % for 15 K of warming | Inside the 4004's +/-5 % with 3 % to spare [CALC] |
| Minimum load | not required | The 1 kohm dummy loads are for measurement only |
| Ripple and noise | 110 mVp-p max, 20 MHz bandwidth | On a 15 V rail, 0.7 %. Not visible on a DMM |
| Transient response | 5 % max deviation, 250 us typ recovery, 25 % load step | 0.75 V dip or bump when the 4004's current steps |
| Capacitive load | 680 uF max on a 15 V single output | The 1 uF and 100 nF on the board are nothing against that |
| Short circuit | continuous, automatic recovery, foldback | Survives a shorted socket pin |
| Current limit | 120 % min, 130 % typ of 67 mA, so 80 to 87 mA | Not a chip protector. The bench supply limit at 60 mA still matters |
| Switching frequency | 220 kHz typ, PFM | |
| Input, no load | 40 mA typ at 5 V in | USB sees this before the chip draws anything |
| Input surge | 15 V max for 1 s | |
| Undervoltage lockout | 4 V max | A sagging USB rail drops the 15 V cleanly rather than browning it out |
| Reflected ripple | 80 mAp-p typ on the input | 100 uF on the 5 V side earns its place |
| Isolation | 1500 VDC, 60 s; 50 pF input to output | Output can float, so the +15 V scheme or a shifted-ground scheme are both available |
| Recommended input fuse | 500 mA slow blow | The MF-R025 polyfuse at 250 mA hold trips earlier; fine for a bench board |
| Temperature | -40 to +95 C operating, case 105 C max | |
| Pinout, single output | 1 -Vin (GND), 2 +Vin, 4 +Vout, 6 -Vout, pins 3 and 5 absent | -Vout ties to board ground for the +15 V scheme |

Conclusions for the rail:

- Regulated to about +/-2 % worst case against a chip that allows +/-5 %. That
  is the confidence number the whole experiment order was for.
- No minimum load means the module can sit on the bench with an empty socket
  and read 15.0 V on the DMM, and the same reading holds with the chip in.
  The ROE-0515S will not do that; measuring how far it moves is the point of
  having one.
- The 5 % transient figure is the one to respect. The 4004's supply current
  changes with instruction activity, and 0.75 V excursions for 250 us at
  each step are inside the chip's window but not by much. The 1 uF ceramics
  at the socket are for that. A scope would show whether more is needed.
- Current limit at 80 to 87 mA does not protect a 4004 that draws 40 mA
  normally. First power-up is still on the bench supply with its limit set to
  60 mA, with the module brought in only after the chip is known good.
- Input side: 40 mA at no load plus about 60 mA of reflected input current at
  full output, from a USB 5 V that also feeds the 74AHCT541 and CD4504B. Well
  inside a 500 mA USB budget. The polyfuse rating (250 mA hold) stays.

Summary of what the datasheets changed:

1. LM2901 at VCC = 5 V with a 14.5 V input is legitimate. My objection in 2.6
   was wrong. The comparator still moves to the 15 V rail, because that is
   the only way to put the reference at 11.5 V.
2. Clock pull-ups are out. BS170 Coss doubles the node capacitance, the
   resistor would have to be 450 ohms, and the current does not fit a 1 W
   module. Clocks go through CD4504B channels, RetroShield style.
3. BS170 gates need 5 V drive. 3.3 V is not enough for a worst-case part.
4. TP0606N3 is guaranteed at a 5 V gate (7 ohms max) and takes the 15 V gate
   swing. Good high-side part.
5. LM319 needs a 2:1 input divider and draws real current. Fallback for the
   bus lines only.

Priced list: `bom/pmos-4004-bom.csv`, Mouser cart
`bom/pmos-4004-order-mouser.csv`. Ordered at Mouser 2026-11-13 after the
datasheet pass in 2.7 (second CD4504B because all six channels are used, two
spare P-FETs, TMR 1-0513 in place of the dual-output 0523): 15 lines, 49.08
USD merchandise, 64.73 USD with shipping, tax and tariff. The 8008 lists
(`bom/8008-archive/`) are kept for when an 8008 turns up.

Parts implied, none ordered: P-channel TO-92 FET for the +5 V side (BS250P,
or ZVP2106A which has a lower on resistance), N-channel TO-92 for the FPGA
side of any bus driver (BS170 or 2N7000). Mouser prices on 2026-11-06 at
quantity one: BS170 0.74, BS250P 1.75, ZVP2106A 1.95, LM2901N 0.66, all in
stock.

---

## 3. Shared section

### 3.1 The two designs side by side

Common to both boards:

- The same RECOM ROE-0515S 5 V to 15 V module provides the extra rail.
- An LED in series with a 5.6 V zener shows that the extra rail is really up.
- The interface chips that drive the CPU are powered from the CPU's own
  positive rail (VCC on the 8008, VSS on the 4004), so an input can never be
  pushed above that rail and the high level tracks it.
- 680 ohm series resistors on the data bus only. Clocks and control inputs are
  driven directly. Status outputs go directly into receiver inputs.
- One MCU pin controls data bus direction. Timing of that pin is firmware.
- Clocks are ordinary MCU outputs through the same buffer type as every other
  input. Neither board has a dedicated clock driver, Schottky diodes or
  discrete transistors.
- No supply sequencing or supervision.

Chip specific:

| | 8008 rev B | 4004 |
|---|---|---|
| Positive rail | +5 V | +15 V |
| Negative rail | -9 V (floating 15 V plus 78L09) | none, VDD is GND |
| DC/DC must be isolated | yes | no |
| Interface logic | 2 x 74LS245 at 5 V | CD40109B and 3 x CD4504B across 5 V and 15 V |
| Level shifting | none, the chip is treated as TTL | true 5 V to 15 V and 15 V to 5 V shifters |
| Pull-downs to VDD on outputs | none | 10k on every output and bus line |
| Negative swing handling | LS input clamp diodes | not needed, nothing goes below GND |

### 3.2 Could one level-shifting design serve both chips?

Partly.

- Receive side: yes. A series resistor, a Schottky clamp to ground and a
  3.3 V buffer with 5 V tolerant inputs works for 8008 outputs and for 4004
  outputs when the 4004 runs from +5 V and -10 V. The 4004 additionally needs
  a pull-down to VDD on each output.
- Drive side: only if you build the 4004 version and accept that it is
  overkill for the 8008. The 8008 takes a TTL low on every input including
  the clocks. The 4004 needs data and control lows at or below -0.5 V and
  clock lows at or below -8.4 V in the +5 V and -10 V arrangement.
- Rails differ: -9 V +/-5 % for the 8008 and -10 V +/-5 % for the 4004. The
  windows only overlap between about -9.25 and -9.45 V [CALC], so make the
  negative rail selectable.
- The alternative for the 4004 is what the RetroShield does: VSS at +15 V,
  VDD at ground, and standard CMOS level shifters. That has nothing in common
  with an 8008 interface apart from the DC/DC module.

My recommendation is separate boards that share the receive-side idea.

Decision recorded 2026-11-06: the 4004 will use Intel's arrangement, VSS at
+5 V and VDD at -10 V, not the RetroShield's +15 V and 0 V. Consequences:

- One supply design serves both chips. The same floating 15 V module takes a
  78L09 for the 8008 or a 78L10 for the 4004.
- The 4004 window is 15 V within 5 %. With +5 V and -10 V each within 5 % the
  sum runs from 14.25 to 15.75 V, which is exactly the allowed range with no
  spare [CALC]. Keep the 5 V rail tight.
- The receive path (series resistor, Schottky clamp, 3.3 V buffer, plus a
  pull-down to VDD on each 4004 output) is shared with the 8008.
- The CD4504B and CD40109B shifters are not used. Driving the 4004 becomes
  the hard part: data and control lows must reach -0.5 V or lower, TEST 0.8 V
  or lower, and both clocks within 1.6 V of -10 V with 50 ns edges. That
  drive circuit is not designed yet. Section 2.6 sketches it: a P-channel
  FET from +5 V plus a 680 ohm pull-down to -10 V per clock, TEST and RESET,
  and a push-pull tri-state driver per bus line.

### 3.3 The hypothesis checked point by point

Clock driver requirements:

- 8008: swing is only "below 0.8 V" to "above VCC - 1.5 V". Edges 50 ns or
  faster into 10 pF. Non-overlap is the table in 1.4 (tD2 at least 0.4 us,
  tD3 at least 0.2 us, tD1 between 0.9 and 1.1 us). A 5 V CMOS buffer with
  TTL-level inputs does it. No negative swing is needed.
- 4004: low must be within 1.6 V of VDD and high within 1.5 V of VSS, so
  nearly the whole 15 V. Edges 50 ns into 20 pF. Width 380 to 480 ns, phi1 to
  phi2 400 to 550 ns, phi2 to phi1 at least 150 ns. This needs a push-pull
  driver running across the full 15 V.
- A diode plus pull-down resistor cannot be a clock driver for either chip. To
  get a 50 ns fall into about 25 pF of pin plus wiring you need roughly
  2.2 x R x C under 50 ns, so R under about 900 ohms, which then burns about
  15 mA whenever the buffer is high [CALC].
- Both chips specify a maximum clock period (3 us and 2 us). They are dynamic,
  and the datasheets do not allow a paused or stretched clock. Use READY to
  stall the 8008.
- In practice both tolerate more than the datasheet says [FW]. The Mega 4004
  sketch runs the 4004 far slower than its minimum rate (see 2.5). The 8008
  sits with both clocks stopped for an arbitrary time before start-up and then
  starts normally through INT (see 1.7), which shows the reset path survives a
  stopped clock but says nothing about register contents surviving one. For a
  lockstep board, stay inside the datasheet limits: an out-of-spec clock is one
  more way for the real chip to diverge from the model.

Pull-down sizing against pin capacitance:

- 8008: not applicable. No pull-downs are needed on inputs or on outputs.
- 4004 inputs with a diode and a pull-down to -10 V, in the +5 V and -10 V
  arrangement: the pin must fall from about 4.7 V to -0.5 V. That takes
  R x C x ln(14.7 / 9.5) = 0.44 x R x C [CALC]. With about 25 pF, 10k gives
  110 ns and costs 1.5 mA while high. 4.7k gives 51 ns and 3.1 mA. Either fits
  inside the 350 ns data setup time. TEST only needs to reach +0.8 V.
- 4004 data bus pull-downs also work against the bus hold time. See the note 3
  discussion in 2.4: with pull-downs the driver has to hold actively through
  the 150 ns after M2 and X2.
- 4004 outputs: Intel used 12k and 5.6k to VDD (Figure 38). The RetroShield
  uses 10k. With 10k and about 40 pF the time constant is 0.4 us, so a passive
  fall alone would be slow against a 1.35 us period. The 4004's own pull-down
  device does part of the work [INFER].

Data bus turnaround and direction control:

- Neither RetroShield derives direction in hardware. Both use one MCU pin.
- 8008: the chip drives in T1, T2 and T3 of a write. The outside world drives
  in T3 of fetch and read cycles. The bus floats in WAIT and STOPPED. Cycle
  type is on D6 and D7 during T2. Details and the T3 enable timing problem are
  in 1.6.
- 4004: fixed schedule by clock count from SYNC (A1 to A3 CPU, M1 and M2
  external, X1 CPU, X2 either, X3 CPU), with a 150 ns bus hold requirement at
  the M2 to X1 and X2 to X3 boundaries.

Noise margin on the "4.7 V high":

- 8008, 5 V HCT buffer on the same rail as VCC, no diode: high is about
  VCC - 0.1 V, need is VCC - 1.5 V, margin about 1.4 V, and it is ratiometric
  [CALC, HCT level from MEM].
- 8008 with the series Schottky: about 0.3 V less, so about 1.1 V. Still fine
  but the diode buys nothing on this chip.
- 8008 as built on the RetroShield with LS245: about 3.4 V typical against a
  3.5 V requirement. No margin [MEM for the LS figure].
- 4004 at VSS = +5 V with HCT plus Schottky: same arithmetic as the 8008,
  about 1.0 to 1.2 V nominal. Worst case with the buffer sourcing several
  milliamps into a low-value pull-down it shrinks to roughly 0.5 V [CALC from
  MEM figures]. Use 10k rather than 4.7k if the timing allows.
- 4004 on the RetroShield data bus: about 0.5 V (14.04 V against 13.5 V).

Power-up sequencing, latch-up and damage:

- The real hard limit on both chips is the same: no pin may be more than 0.5 V
  above the positive rail (VCC or VSS) and nothing may be more than 20 V below
  it [DS 8008UM p.14, DS 4004 p.7]. So never drive an input while the chip's
  positive rail is off. Powering the driver from the chip's positive rail, as
  both RetroShields do, makes that automatic.
- Both chips are single-channel PMOS. There is no parasitic CMOS thyristor in
  them, so classic latch-up is not the concern. The concern is forward-biasing
  a pin junction above the positive rail [INFER].
- The CMOS chips around them can latch up. An 8008 or 4004 output pulling an
  HCT or LVC pin below ground injects current into that chip's substrate
  diode. Limit it with a series resistor and take it in an external Schottky
  [INFER].
- 8008 power-up reset depends on the chip "sensing the rise of VDD"
  [DS 8008UM p.11]. Nothing in the pages I read says how fast or in what order
  the rails must rise. See open questions.
- 8008 negative rail: if the DC/DC is not isolated the 78L09 arrangement is a
  short. If the 78L09 fails shorted, VDD becomes about -15 V and VCC - VDD
  reaches the 20 V absolute maximum [CALC].
- CD4504B and CD40109B are specified as independent of supply sequencing, and
  VCC may exceed VDD [DS cd4504b.pdf p.1, cd40109b.pdf p.1].
- 4004 board: an unloaded ROE-0515S can exceed 15 V +5 % [INFER]. The CD4000
  parts are rated to 18 V operating, 20 V absolute [DS cd4504b.pdf p.1, p.3].
  The 4004's own absolute limit is 20 V.

### 3.4 What changes going from a 5 V Arduino Mega to a 3.3 V FPGA

- A 3.3 V output cannot reach the 8008's 3.5 V VIH. Every signal into the 8008
  must pass through a 5 V powered buffer that accepts 3.3 V inputs (74HCT or
  74AHCT, VIH 2.0 V [MEM]). This also fixes the RetroShield's missing VIH
  margin.
- ECP5 I/O is not 5 V tolerant [MEM, check the ECP5 datasheet]. Every signal
  from the chip must pass through a 3.3 V powered buffer with 5 V tolerant
  inputs (74LVC [MEM]).
- LS-TTL inputs shrug off being pulled below ground. CMOS inputs need a series
  resistor and an explicit Schottky clamp.
- An FPGA can and should meet the real clock specification. Once it does, the
  8008's output delays (up to 1.1 us) and the T3 enable timing become real
  design constraints. A bit-banged Arduino never had to face them.
- The RetroShield already has a 3.3 V host, the Teensy 4.1, and it gets to
  5 V through TXS0108 auto-direction translators on the adapter board
  (`hardware/teensy`) in front of the same 74LS245s. That does nothing for the
  8008's VIH, and auto-direction translators have weak drive, which is
  presumably why the 8008 schematic says "Teensy can not drive LED's" and
  buffers its LED [INFER]. I would not copy that arrangement for an FPGA that
  needs deterministic timing.
- FPGA pins float during configuration. Put pull-downs on the buffer inputs
  for phi1, phi2 and INT, and a pull-up on the data driver's output enable so
  it is off by default.
- For the 4004 RetroShield scheme: the CD4504B clock and control shifter must
  have SELECT tied high (TTL mode, VIH 2.0 V at VCC = 5 V) to accept 3.3 V
  [DS cd4504b.pdf p.2]. The CD40109B needs 0.7 x VCC, so 3.5 V at VCC = 5 V
  [DS cd40109b.pdf p.3], which a 3.3 V output does not reach. The down-shifter
  outputs are 5 V and need a further step to 3.3 V.

### 3.5 What I got wrong

"I" here is the original hypothesis. Items are ordered by how much they change
the design.

1. The RetroShield 4004 does not put VSS at +5 V and VDD at -10 V. It puts VSS
   at +15 V and VDD at ground and uses CD40109B and CD4504B level shifters.
   There is no negative rail on that board. The +5 V and -10 V arrangement is
   the one Intel documents for TTL compatibility, so the hypothesis is a valid
   design, just not this one.
2. The RetroShield 8008 has no 74HCT buffers, no Schottky diodes and no
   pull-downs. It is two 74LS245s, eight 680 ohm resistors and a -9 V supply.
3. The diode and pull-down circuit is unnecessary for the 8008. Its inputs,
   clocks included, accept a low of up to 0.8 V. The part of the 8008 input
   spec that is hard to meet is the high level, 3.5 V, not the low level.
4. 8008 outputs do not need pull-downs to VDD. They are push-pull between VCC
   and VDD and pull themselves toward -9 V. The thing to add is a limit on how
   far below ground the receiving pin is dragged.
5. 4004 clock inputs have a much lower VIL than the other inputs: VSS - 13.4 V
   against VSS - 5.5 V. A circuit sized for the data pins will not drive the
   clocks.
6. 4004 TEST has its own VIL, VSS - 4.2 V.
7. The 8008 has VCC and VDD only. There is no VSS pin.
8. The 4004 thresholds quoted (VIH VSS - 1.5 V, VIL VSS - 5.5 V) are correct
   for the non-clock inputs.
9. The hypothesis that 4004 outputs need pull-downs to VDD is correct. Intel
   says so and the RetroShield fits 10k on every one.
10. Missing from the hypothesis: both chips have a maximum clock period, so
    the clock cannot be stopped. The 8008 has no reset pin and must be started
    with INT. The 8008 data bus floats in WAIT and STOPPED. The ROE-0515S has
    no current headroom at the 8008's maximum IDD.

### 3.6 Open questions

1. Answered by 1.7: predictive direction control and an in-spec clock near
   2.1 us from a Teensy 4.1. There is no Arduino Mega firmware for the 8008 in
   any of the three repos, so "how does the Mega do it" has no answer to find.
2. How far below ground does a lightly loaded 8008 output go? The manual gives
   no VOL minimum. Measure it before trusting any clamp current estimate.
3. Mostly answered by 1.7: the RetroShield lets VDD come up with the clocks
   stopped, waits an arbitrary time, and the INT start-up still works. A slow
   VDD ramp is still untested. An FPGA-controlled VDD enable is now a
   convenience for experiments, not a requirement.
4. Are the 8008 flags defined after the power-up clear? The manual lists the
   accumulator, scratch pad, program counter and stack only.
5. Exactly when may external data start and stop in T3? The firmware's answer
   is "start 400 ns after phi22 of T2 falls, stop when SYNC falls", and it
   works with 680 ohm resistors. The earliest safe start without contention is
   still unmeasured.
6. Answered: yes in at least one case. For INP the flags come out at T4. Do
   not drive in T4 or T5.
7. Is the phi1 width minimum 0.70 us (AC table) or is 0.50 us acceptable
   (SIM8-01 figure)? The firmware uses 0.70 us for the 8008 and 0.35 us for
   the 8008-1, following the AC table. Scott Baker's boards run eight CPUs at
   0.50 us (1.8). Both work. Design to the table.
8. What did the earlier k8008 revision look like? Firmware comments mention a
   rev A with inverted clocks and a level shifter in the data path. No
   schematic for it is in either repo.
9. What is the 5 V position of J2 on the 4004 board for?
10. Which 8008 will be on the breakout, 8008 or 8008-1? The clock limits
    differ.
11. 4004 bus driver (2.6): push-pull tri-state per line, or a switched
    pull-down like the SIM4 diode disconnects? Neither is designed.
12. Answered in 2.7: BS170 is only guaranteed at a 10 V gate, typical curve
    gives about 3 ohms at 5 V, so 5 V drive is fine and 3.3 V is not. TP0606
    replaces the BS250 and is guaranteed 7 ohms at a 5 V gate.

### 3.7 Numbers that are still unverified

- LM2901 common-mode behaviour, response time, BS170 threshold and TP0606
  figures: now verified, see 2.7. LM319 numbers were then confirmed against
  the TI LM119/LM219/LM319 datasheet (SNOSBJ2): differential input +/-5 V
  absolute maximum, input voltage equal to the supply below +/-15 V, 80 ns
  typical, positive supply current 8 mA typical and 12.5 mA maximum at
  +/-15 V. TP0606 numbers confirmed against the DigiKey-hosted copy. Both
  found through `dkp part`, which returns a datasheet link per part.
- TMR 1-0513 datasheet: read 2026-11-13 from the downloaded copy,
  see 2.7. Nothing left unverified on the rail except what only a scope can
  see (ripple, transient excursions, clock edges).

- 74LS245 VOH (2.4 V guaranteed, about 3.4 V typical) and input clamp rating.
- 74HCT541 output levels and input thresholds.
- 74LVC245 input tolerance, thresholds and absolute minimum input voltage.
- BAT54 forward voltage at 1 to 2 mA.
- ECP5 I/O voltage limits.
- Every [INFER] and [MEM] item above.
- The CD4504B thresholds at VCC = 15 V. The datasheet table stops at 10 V.
- The MCS-4 manual was keyword searched, not read through. Its Appendix A
  electrical tables were not compared line by line with the later 4004
  datasheet.
- The 78L09 figures come from a keyword search of its datasheet. I did not
  confirm which table row applies to the 9 V part by reading the page.
- All 8008 timing figures were read from one scanned manual. Intel issued
  several revisions. A Rev 4 (Nov 73) scan is in the firmware repo and its
  tables were not compared.
- Firmware timings in 1.7 are the values written in the source. Nothing was
  measured, and the NOP-based delays depend on the Teensy's clock speed.
- `hardware/teensy` was read for its part list only. How each TXS0108 channel
  is wired was not traced.
- In `8008-super` the DC/DC module and zener part numbers are not on the
  schematic, the Master sheets were read as text only, and only two of the
  sixteen captures were looked at. The clock widths there are inferred from
  the logic and the captures, not from a stated figure.

### 3.8 Recommended starting circuit for the 8008 board

Rails:

- +5 V feeds 8008 VCC (pin 10) and the 5 V buffers, from the same net.
- -9 V on VDD (pin 1). The floating 15 V plus 78L09 arrangement from the
  RetroShield is fine. Use an isolated 5 V to 15 V module with more than 66 mA
  so that 60 mA of IDD plus the regulator leaves headroom.
- Bypass VCC to GND and VDD to GND close to the socket, and put one capacitor
  directly from VCC to VDD at the chip. Scott Baker had to add that one in a
  later revision (1.8).
- Whatever makes the negative rail, give it a minimum load and a clamp. Both
  prior designs use unregulated isolated modules, which rise when lightly
  loaded. A simpler alternative to the 78L09 arrangement is an isolated 5 V to
  9 V module with its positive output grounded and a 9 V zener across it
  (1.8). It has more current headroom and looser regulation. With donated
  chips I would keep the regulator.
- Optionally give the FPGA a way to switch VDD on. The RetroShield manages
  without one (see 1.7), but it makes power-up experiments repeatable.

FPGA to 8008 (phi1, phi2, READY, INT):

- FPGA pin to 74HCT541 input. 74HCT541 powered from the 8008's +5 V.
- 74HCT541 output through 33 ohms to the 8008 pin. No diode, no pull-down.
  A 1k pull-up to +5 V on each of these pins, as on Scott Baker's boards and
  as Intel recommends, is optional with a rail-to-rail driver but harmless.
- INT must leave the FPGA registered on the rising edge of phi1.
- 10k pull-down on the buffer inputs for phi1, phi2 and INT. Pull READY
  whichever way suits bring-up.

Data bus, per bit:

- 8008 pin, then a 470 ohm series resistor, then a node N.
- On N: the output of a second 74HCT541 (FPGA to 8008), a BAT54 Schottky with
  anode to GND and cathode to N, and the input of a 74LVC245 or 74LVC541
  powered from 3.3 V (8008 to FPGA).
- 100k from N to +5 V so the bus reads a defined level when it floats.
- The data 74HCT541 output enable is driven by the FPGA with a pull-up to +5 V
  so it is off by default.

8008 to FPGA (S0, S1, S2, SYNC):

- 8008 pin, 1k series resistor, BAT54 to GND, 74LVC541 input at 3.3 V.

Checks to make on the first board:

- Voltage at node N and at the 8008 pin while the 8008 drives low. Confirm N
  stays above -0.4 V and note the current.
- phi1 and phi2 levels and edges at the socket.
- The T3 data window from 1.6.

FPGA logic:

- Generate phi1 and phi2 from one phase counter using the 44-tick schedule in
  1.6 and move to a faster schedule only after the board works.
- Advance the soft core from the same counter.
- Sample status and data at tick 28 to 30 of the second period of each state.
- Run the start-up sequence from 1.6 on both cores. The firmware's version
  (40 clock periods, then INT, jam 0x05) is a known-good reference.
- Anticipate T3 at the end of T2, as Intel's SIM8-01 and the firmware both do.

An alternative for the data bus is a single dual-supply transceiver with TTL
thresholds on its 5 V side (74LVC4245 class [MEM]). Avoid transceivers whose
5 V side has CMOS thresholds (0.7 x VCC is 3.5 V), because the 8008 only
guarantees 3.5 V and there would be no margin.

---

## Appendix A. 4004 RetroShield netlist

From the KiCad conversion of `hardware/k4004s/k4004s.pcb`. Format is
`refdes.pad`. J1 pad numbers follow the gEDA symbol: 1 to 18 down the left
column, 36 down to 19 on the right column.

```
+5V          C3.2 C6.2 C7.2 C8.2 J1.1 J1.36 J2.3 U2.1 U3.16 U4.1 U5.16 U9.2
GND          C3.1 C4.1 C6.1 C7.1 C8.1 C9.1 C10.1 C11.1 D1.2 J1.18 J1.19
             Q0.2 Q1.2 Q2.2 Q3.2 R3.1 R10.1 R26.2 R27.2 R28.2 R29.2 R30.2
             R31.2 R32.2 R33.2 R34.2 R35.2 U1.12 U2.8 U3.3 U3.8 U3.13 U3.14
             U4.8 U4.9 U4.11 U4.13 U5.8 U5.13 U9.1 U9.3
P15V         C9.2 C10.2 C11.2 J2.2 R20.2 U1.5 U2.16 U3.1 U4.16 U5.1
(15 V raw)   C4.2 J2.1 R10.2 U9.4
D0_15V       U1.1 R8.2 R35.1 U3.9
D1_15V       U1.2 R7.2 R34.1 U3.7
D2_15V       U1.3 R9.2 R33.1 U3.11
D3_15V       U1.4 R6.2 R32.1 U3.5
(U2 outputs) U2.11-R8.1  U2.5-R7.1  U2.13-R9.1  U2.4-R6.1
ARDUINO_D0   J1.28 U2.10      ARDUINO_D1   J1.9  U2.6
ARDUINO_D2   J1.29 U2.14      ARDUINO_D3   J1.8  U2.3
DRIVE_DBUS   J1.10 R3.2 U2.2 U2.7 U2.9 U2.15
CPU_D0_5V    J1.22 U3.10      CPU_D1_5V    J1.15 U3.6
CPU_D2_5V    J1.23 U3.12      CPU_D3_5V    J1.14 U3.4
PHI1         J1.11 U4.3       PHI1_15V     U4.2  U1.6
PHI2         J1.26 U4.14      PHI2_15V     U4.15 U1.7
RESET        J1.17 U4.7       RESET_15V    U4.6  U1.9
TEST         J1.16 U4.5       TEST_15V     U4.4  U1.10
SYNC_4004    U1.8  R31.1 U5.3     SYNC_5V    U5.2  J1.33
CMROM_4004   U1.11 R30.1 U5.14    CMROM_5V   U5.15 J1.4
CMRAM0_4004  U1.16 R29.1 U5.9     CMRAM0_5V  U5.10 J1.2
CMRAM1_4004  U1.15 R28.1 U5.7     CMRAM1_5V  U5.6  J1.35
CMRAM2_4004  U1.14 R27.1 U5.11    CMRAM2_5V  U5.12 J1.3
CMRAM3_4004  U1.13 R26.1 U5.5     CMRAM3_5V  U5.4  J1.34
RAM_Q0_5V    J1.30 R12.1 (R12.2-Q0.1)   RAM_Q1_5V  J1.7  R13.1 (R13.2-Q1.1)
RAM_Q2_5V    J1.31 R14.1 (R14.2-Q2.1)   RAM_Q3_5V  J1.6  R15.1 (R15.2-Q3.1)
P15V LED     R20.1-P15V.1  P15V.2-D1.1
unconnected  J1.5 J1.12 J1.13 J1.20 J1.21 J1.24 J1.25 J1.27 J1.32
             U2.12 U3.2 U3.15 U4.10 U4.12
```

Values: R3, R10, R26 to R35 10k. R6 to R9 and R12 to R15 680. R20 3.1k.
C3, C4 4.7 uF. C6 to C11 100 nF. D1 5.6 V zener. U1 4004, U2 CD40109B,
U3, U4, U5 CD4504B, U9 ROE-0515S.

## Appendix B. 8008 RetroShield rev B netlist

Transcribed by hand from the schematic PDF. J1 pad numbers alternate left and
right (odd left, even right). The header assignments were cross-checked
against the firmware pin map (`t8008_intellec_8/portmap.h`: CLK1 PC3, CLK2 PC2,
INTERRUPT PC1, READY PC0, SYNC PC4, S0 PC5, S1 PC6, S2 PC7, data PL0 to PL7,
buffer direction PG0, reset key PG2, debug LED PA7) and agree. The 4004 header
assignments in Appendix A agree with `t4004_empty/portmap.h` in the same way.
The connections between the buffers and the 8008 itself have no second source.

```
VCC    J1.1 J1.2 U1.10 U2.20 U3.20 U3.1 U4.2 U6.5 C1 R10
GND    J1.35 J1.36 U2.10 U2.19 U3.10 U3.19 U4.1 U5.1(OUT) U6.3 C1 C2
       R9.1 PWR-LED anode, RESET switch pins 3 and 4
VDD    U1.1 U4.3 U5.2 C2 C3 D1.3            (net NEG15V, -9 V)
POS15V U4.4 U5.3 C3
D7_MEGA..D0_MEGA   J1.23,24,25,26,27,28,29,30 -> U2.2..U2.9 (A1..A8)
U2 B1..B8 (18..11) -> R7..R0 pin 1 ; R7..R0 pin 2 -> D7_15V..D0_15V -> U1.2..U1.9
DATABUF_DIR_MEGA   J1.22 U2.1
S2_15V   U1.11 U3.2     S2_MEGA   U3.18 J1.11
S1_15V   U1.12 U3.3     S1_MEGA   U3.17 J1.12
S0_15V   U1.13 U3.4     S0_MEGA   U3.16 J1.13
SYNC_15V U1.14 U3.5     SYNC_MEGA U3.15 J1.14
CLK2_MEGA  J1.16 U3.6   CLK2_15V  U3.14 U1.15
CLK1_MEGA  J1.15 U3.7   CLK1_15V  U3.13 U1.16
READY_MEGA J1.18 U3.8   READY_15V U3.12 U1.17
INT_MEGA   J1.17 U3.9   INT_15V   U3.11 U1.18
LED_DBG    J1.10 U6.2 ; U6.4 -> DBG LED -> R9 -> GND
KEY_RESET  J1.20 R10 RESET switch pins 1 and 2
PWR LED    GND -> LED -> R8 (1.1k) -> D1.2 ; D1.3 -> VDD ; D1.1 unconnected
unconnected J1.3 to J1.9, J1.19, J1.21, J1.31 to J1.34
```

## Appendix C. KiCad conversion status

| Item | Status |
|---|---|
| 4004 board | Converted. `kicad-cli pcb import --format auto hardware/k4004s/k4004s.pcb` (KiCad 10.0.4) recognised it as gEDA / Lepton EDA: 43 footprints, 579 tracks, 87 vias. Output is `reference/retroshield-k4004s-kicad/k4004s.kicad_pcb` in this project. Not opened in the GUI or DRC checked |
| 4004 schematic | Not converted. KiCad 10 has a gEDA schematic importer but `kicad-cli` does not expose it ("Failed to load schematic"). It has to be done in the GUI: schematic editor, File, Import, Non-KiCad Schematic, choose `hardware/k4004s/k4004s.sch`. The importer reads `gafrc` to find `./symbols` |
| 8008 schematic and board | Not converted. DipTrace `.dch` and `.dip` are binary and KiCad cannot read them. DipTrace 5.2.0.4 is installed on this machine and can export Eagle XML or DipTrace ASCII from its File, Export menu in both the schematic and PCB editors. KiCad imports Eagle XML (`kicad-cli pcb import --format eagle` for the board, GUI import for the schematic). This needs a person at the DipTrace GUI |
