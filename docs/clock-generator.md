# 4004 clock generator in TTL

Two-phase non-overlapping clock for the 4004, built from parts on the shelf, to be understood on the scope before any host logic exists. The host (FPGA or Pico) takes this job over later; until then this board is the clock.

Sources: Intel 4004 datasheet pp. 8-21 to 8-23 [DS]; TI SDLS060 (SN74LS163A, SN74S163) [DS]; TI SDLS025D (SN74LS00, SN74S00) [DS]; TI SCHS069D (CD4504B) [DS]. [CALC] is my arithmetic, [FIG] a reading of a figure.

## What the 4004 requires

```
φ1  ‾‾‾‾‾‾‾‾\______/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\______/‾‾‾
φ2  ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\______/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
             |<-tφ->|<--tD1-->|<-tφ->|<-tD2->|
             |<------------- tCY ------------>|
```

| Parameter | Min | Max | This design at the 50 % crossings | At the datasheet's reference points, typ / max CD4504B edge |
|---|---|---|---|---|
| tCY, period | 1.35 µs | 2.0 µs | 1.956 µs | 1.956 µs |
| tφPW, pulse width, low | 380 ns | 480 ns | 489 ns | 449 / 409 ns |
| tφD1, φ1 trailing edge to φ2 leading edge | 400 ns | 550 ns | 489 ns | 449 / 409 ns |
| tφD2, φ2 trailing edge to φ1 leading edge | 150 ns | | 489 ns | 449 / 409 ns |
| tφR, tφF, 10 to 90 % | | 50 ns | | CD4504B 40 typ, 80 max into 50 pF |
| VIHC at the pin | 13.5 V | 15.3 V | | CD4504B VOH 14.95 V min |
| VILC at the pin | 0 V | 1.6 V | | CD4504B VOL 0.05 V max |

Both phases rest high (at VSS, 15 V at the pin) and pulse low. The 4004 is dynamic logic: the clocks never stop while VSS is present with a chip in the socket.

### Where the datasheet measures

Figure 2 on p. 8-23 [FIG]: rise and fall are 10 to 90 %; **tφPW is measured at the 90 % level**, fully-low to fully-low; **tφD1 and tφD2 are measured at the 10 % level**, from the top of one edge to the top of the next. tφD1 runs from φ1's rising edge to φ2's falling edge, a trailing-edge-to-leading-edge gap. With an edge time t_e, both the width and the gaps measured this way come out **one full edge time shorter** than the same intervals at 50 % [CALC].

A design with four equal ticks T must therefore satisfy:

- width: 380 ≤ T − t_e ≤ 480
- tφD1: T − t_e ≥ 400
- period: 4T ≤ 2000 ns

For the CD4504B (t_e 40 to 80 ns) the window is T from 480 to 500 ns. **The upper limit depends on the driver.** With a fast driver (t_e under 10 ns) T must come down to 480 or below; see Alternatives.

## Design

T = 488.9 ns from the 24.545 MHz can divided by 12 in two stages, neither of which has a feedback path that is tight at its clock rate.

```
24.545 MHz can (ECLIPTEK EC1125 on the foam; confirm 5 V logic output on the scope)
   │
   ▼
SN74S163J free-running, QB = ÷4 = 6.136 MHz (163 ns period)       no feedback
   │
   ▼
DM74LS163AN counting 0, 1, 2 (mod 3): /CLR = NOT(QB), QB out = 2.045 MHz tick   feedback at 163 ns, needs 62 ns max
   │
   ▼
DM74LS74AN as a 2-stage Johnson counter: states (QA,QB) 00 → 10 → 11 → 01 → 00
   │
   ▼
SN74LS00N: /φ1 = NAND(/QA, /QB) low only in state 00; /φ2 = NAND(QA, QB) low only in state 11
   │
   ▼
CD4504B channels A and B, SELECT high (TTL input mode), VDD = 15 V
   │
   ▼
4004 pin 6 (φ1), pin 7 (φ2), direct, no series resistor
```

Why a Johnson counter: only one flip-flop changes per tick, so each decode NAND sees one input move at a time and cannot glitch. A binary counter decoded with gates can produce a few-nanosecond spike where two bits change together, and a spike on a clock pin is a spurious clock.

### Why two counter stages

Any modulus that is not a power of two needs a feedback path: a decode of the terminal count back into a synchronous LOAD or CLR, and the whole path must settle inside one clock period. From SDLS060 and SDLS025D, maxima:

| Path element | 74S163 | 74LS163A |
|---|---|---|
| CLK to RCO | 25 ns | 35 ns |
| CLK to any Q | 15 ns | 27 ns |
| /LOAD setup | 14 ns | 20 ns |
| /CLR setup | 14 ns | 20 ns |
| recommended clock frequency, max | 40 MHz | 25 MHz |
| 74LS00 gate, max | 15 ns | |
| 74S00 or 74S10 gate, max | 5 ns | |

A mod-13 S163 clocked at 26.824 MHz (37.3 ns period) via RCO needs 25 + 15 + 14 = 54 ns with an LS inverter, 44 ns with an S inverter. Neither fits. Even the typical sum (14 + 9 + 14 = 37 ns) is the whole period. That design is withdrawn.

The two-stage version puts the only feedback at 6.136 MHz: LS163A CLK-to-Q 27 + LS00 inverter 15 + /CLR setup 20 = 62 ns against a 163 ns period. 101 ns of slack. The first stage is a free-running divider with no feedback, clocked at 24.5 MHz, inside the S163's 40 MHz rating.

## Wiring

All chips: 0.1 µF ceramic across VCC and GND, right at the pins. 24.5 MHz exists only between the can and S163 pin 2; keep that run short.

### Oscillator

24.545 MHz can, full-size 14-pin footprint: pin 1 no connect (or enable), pin 7 GND, pin 8 output, pin 14 5 V. Output to S163 pin 2 only. Check it on the scope first: 5 V swing, 24.5 MHz.

### SN74S163J, divide by 4

| Pin | Connect to |
|---|---|
| 16 VCC, 8 GND | 5 V, GND |
| 2 CLK | 24.545 MHz can pin 8 |
| 1 /CLR, 9 /LOAD, 7 ENP, 10 ENT | 5 V (count forever) |
| 3, 4, 5, 6 (A, B, C, D) | GND |
| 13 QB | 6.136 MHz, 50 % duty, to LS163A pin 2 |
| 14 QA, 12 QC, 11 QD, 15 RCO | open |

### DM74LS163AN, modulo 3

Synchronous clear: /CLR low at a clock edge sets the count to zero on that edge. QB is high only at count 2, so /CLR = NOT(QB) gives the sequence 0, 1, 2, 0. Any glitch on the decode between edges is ignored because the clear is synchronous.

| Pin | Connect to |
|---|---|
| 16 VCC, 8 GND | 5 V, GND |
| 2 CLK | S163 pin 13 |
| 9 /LOAD, 7 ENP, 10 ENT | 5 V |
| 3, 4, 5, 6 (A, B, C, D) | GND |
| 13 QB | → 74LS00 pins 12 and 13 (spare gate wired as an inverter), **and** the tick to 74LS74 pins 3 and 11 |
| 1 /CLR | ← 74LS00 pin 11 |
| 14 QA, 12 QC, 11 QD, 15 RCO | open |

QB is a 2.045 MHz pulse, high for one 163 ns state in three. Its rising edge is the tick. The 74LS74 needs a clock pulse of 25 ns or more; 163 ns is fine.

### DM74LS74AN, Johnson counter

FF A is pins 1 to 6, FF B is pins 8 to 13.

| Pin | Connect to |
|---|---|
| 14 VCC, 7 GND | 5 V, GND |
| 3 CLK A and 11 CLK B | tick from LS163A pin 13 |
| 2 D A | from pin 8 (/QB) |
| 12 D B | from pin 5 (QA) |
| 1 /PRE A, 4 /CLR A, 10 /PRE B, 13 /CLR B | 5 V |
| 5 QA, 6 /QA, 9 QB, 8 /QB | to the 74LS00 |

### SN74LS00N, decode and the mod-3 inverter

| Pin | Connect to |
|---|---|
| 14 VCC, 7 GND | 5 V, GND |
| 1, 2 | /QA (74LS74 pin 6), /QB (74LS74 pin 8) |
| 3 | /φ1 → CD4504B pin 3 |
| 4, 5 | QA (74LS74 pin 5), QB (74LS74 pin 9) |
| 6 | /φ2 → CD4504B pin 5 |
| 12, 13 | QB of the LS163A (pin 13) |
| 11 | → LS163A pin 1 /CLR |
| 9, 10 | 5 V (unused gate) |

### CD4504B, level shift

| Pin | Connect to |
|---|---|
| 1 VCC | 5 V, 0.1 µF to GND |
| 16 VDD | 15 V, 0.1 µF to GND |
| 8 VSS | GND |
| 13 SELECT | 5 V (TTL-to-CMOS mode, 2.0 V input threshold, so LS can drive it) |
| 3 A in | /φ1 |
| 2 A out | 4004 pin 6, φ1 |
| 5 B in | /φ2 |
| 4 B out | 4004 pin 7, φ2 |
| 7, 9, 11, 14 | the other four inputs; on the full board these are the P-FET gate drives, on the clock breadboard tie them to GND |

The CD4504B is non-inverting. A TTL low in gives about 0.05 V out, a TTL high gives about 14.95 V. The pin sees the 74LS00 waveform at 15 V.

## Expected waveforms

```
tick:       0        1        2        3        0
            |<489ns>|<489ns>|<489ns>|<489ns>|
QA        __/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\________________/‾‾
QB        ___________/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\__________
/φ1 (pin 6) ___/‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\______
/φ2 (pin 7) ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾\________/‾‾‾‾‾‾‾‾‾‾‾
```

Period 1.956 µs at the 50 % crossings. φ1 low in state 00, φ2 low in state 11, one empty state between them each way. At the datasheet's reference points each width and gap reads about 449 ns with a typical CD4504B edge.

CD4504B mismatch budget [CALC]. On one channel the low pulse starts with a high-to-low transition (tPHL) and ends with a low-to-high one (tPLH), so width = 489 − t_e + (tPLH − tPHL). Across channels the gap tφD1 starts with φ1's rising edge through channel A (tPLH_A) and ends with φ2's falling edge through channel B (tPHL_B), so tφD1 = 489 − t_e − (tPLH_A − tPHL_B): a late φ1 rise shrinks the gap.

| | typical edge (40 ns) | maximum edge (80 ns) |
|---|---|---|
| width 380 to 480: tPLH − tPHL on one channel | −69 to +31 ns | −29 to +71 ns |
| tφD1 ≥ 400: tPLH_A − tPHL_B | ≤ +49 ns | ≤ +9 ns |
| tφD1 ≤ 550: tPLH_A − tPHL_B | ≥ −101 ns | ≥ −141 ns |

The CD4504B specifies tPHL and tPLH identically (140 typ, 280 max, TTL mode, 15 V, 50 pF) and does not specify channel skew; the measurement below is what proves it for the parts in the sockets. tφD1 depends on tPLH_A − tPHL_B and tφD2 on tPLH_B − tPHL_A. If tφD1 measures short, swapping which CD4504B channel carries which phase exchanges the two terms, moving the offending skew onto tφD2, which has about 300 ns of margin (449 against a 150 ns minimum) instead of 9 to 49 ns. Re-measure both gaps after a swap.

## Scope checklist, in order

1. Can pin 8: 24.5 MHz, 5 V swing. If the breadboard cannot show it clean, shorten the wire to the S163 or move the can next to it.
2. S163 pin 13: 6.136 MHz square, 50 % duty.
3. LS163A pin 13: 2.045 MHz, high 163 ns, low 326 ns. If it shows 383 kHz (÷16) the clear is not reaching pin 1; if 3.07 MHz (÷2) the clear is stuck low.
4. 74LS74 pins 5 and 9: two 1.02 MHz squares, one lagging the other by a quarter period.
5. 74LS00 pins 3 and 6: 489 ns low pulses, never low together, 489 ns between the end of one and the start of the other.
6. CD4504B pins 2 and 4, **both pins, each of the following**, measured the way the datasheet measures, with the probe as the only extra load:
   - tφPW at the 90 % level: 380 to 480 ns. Expect about 449.
   - tφD1, 10 % point of φ1 rising to 10 % point of φ2 falling: 400 ns or more. Expect about 449.
   - tφD2, 10 % point of φ2 rising to 10 % point of φ1 falling: 150 ns or more. Expect about 449.
   - rise and fall, 10 to 90 %: 50 ns or less.
   - peak: never above 15.3 V. 47 Ω in series at the CD4504B output if there is overshoot.
   Width on each pin separately catches a tPLH/tPHL mismatch on that channel; tφD1 catches channel-to-channel skew.
7. Leave it running an hour and repeat step 6.

## Power-up order with a chip in the socket

5 V on, which starts the can, the counters and the CD4504B input side. Confirm /φ1 and /φ2 pulsing at the CD4504B inputs. RESET asserted (4004 pin 9 at ground) by the default-state resistors described in the tutorial, not by anything the host has to do. Then 15 V on. The clocks are already running the moment VSS appears. Hold RESET for at least 64 clock periods after 15 V is up, then release deliberately from the host. Power down in reverse: 15 V off first, then 5 V. Clocks never stop while 15 V is present.

## Alternatives, recorded

- **36 MHz ÷ 16, T = 444 ns (first version):** at the 50 % crossings it looks comfortable; at the datasheet's 90 % and 10 % reference points the width and tφD1 read 404 ns with a typical CD4504B edge and 364 ns with a maximum one. Replaced.
- **26.824 MHz mod 13 in one S163, T = 485 ns (second version):** right tick, impossible feedback path. CLK-to-RCO 25 + inverter 15 (LS) or 5 (S) + /LOAD setup 14 = 54 or 44 ns against a 37.3 ns clock period. Withdrawn.
- **24.545 MHz mod 12 in one S163 with a 74S10:** decode count 11 (QA, QB, QD) with one 3-input S10 gate into /CLR. CLK-to-Q 15 + S10 5 + /CLR setup 14 = 34 ns against 40.7 ns. Fits with 6.7 ns to spare, one chip fewer than the chosen design. Kept as the compact option; the two-stage version was chosen for its 101 ns of slack.
- **Ticks between 400 and 480 ns** (14.318 MHz mod 6, 23.04 MHz ÷ 10, 29.4912 MHz ÷ 12) all lose their margin once the CD4504B edge time is subtracted. Only ticks of 480 to 500 ns pass at the maximum edge.
- **If the 15 V driver changes to a fast one** (TC4427 class with 47 to 100 Ω series, edges of 5 to 10 ns; with its GND at 0 V and VDD at 15 V its inputs are TTL-compatible, so the 74LS00 drives it directly), the width upper limit bites: T must be 480 + t_e or less. The 36 MHz ÷ 16 design at 444 ns becomes the right answer again with fast edges (width 436, tφD1 436). Retune T to the driver; do not carry 489 ns across.
- **Edge time fallback:** if a CD4504B measures over 50 ns at the pin, that fast driver is the candidate. Datasheet not yet read.
- **Loose two-pin crystal instead of a can:** Pierce oscillator around one CD4049AE inverter, 1 M feedback, 22 pF each leg to ground, series resistor 470 Ω to 2.2 k depending on frequency, second inverter as buffer. Crystals above about 20 MHz are third-overtone and will run at one third of the marked frequency in this circuit.
- **The historical way:** one-shots setting each width with an RC. The HP MCS-4 exerciser does this with three DM9602N; the shelf has the same part. Worth breadboarding once to understand his board, not what this project uses.
