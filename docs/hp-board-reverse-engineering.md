# HP MCS-4 Exerciser COS-2: reverse engineering plan

the boards' owner has no documentation for this board. The board is the only source. Goal: a complete schematic and a parts list with every part identified, before any power. All work power off, with a multimeter, a magnifier and good light. Same method afterwards for the Pro-Log card.

## 0. Record before touching

- Photograph both sides, square on, high resolution, good lighting. Also each IC, transistor and passive close up for markings.
- Build a reference designator map. The silkscreen has U1 to U7 and Q1 to Q10; give every unlabelled part a designator (R1.., C1.., D1..) on a printout of the photo.
- Note the edge test points by name: ground symbol, +5V, DATA 0 to 3, CM-RAM 0 to 3, CM-ROM, TEST, U4-10, S.PASS, COM, SYNC, CLOCK φ1, φ2.

## 1. Known parts first

Pinouts from datasheets, so they anchor the tracing:

| Ref | Part | Datasheet source |
|---|---|---|
| U4 | C4004 | Intel 4004 datasheet ([bitsavers](http://www.bitsavers.org/components/intel/MCS4/)) |
| U5 | P4001-0009 | MCS-4 User's Manual, Feb 1973 ([bitsavers](http://www.bitsavers.org/components/intel/MCS4/)) |
| U6, U7 | P4002-1 | MCS-4 User's Manual |
| U1, U2, U3 | 9602 dual one-shot (Fairchild, F7328) | Fairchild 9602 / National DM9602; the shelf has DM9602N to compare |

MOS supply pins to trace from: 4004 pin 5 VSS, pin 12 VDD. 4001 and 4002: confirm VSS and VDD pin numbers from the manual before probing (16-pin, VSS and VDD are not in the same positions as the 4004). 9602: pin 16 VCC, pin 8 GND.

## 2. Supply nets

Continuity mode, power off, all chips in place is fine for continuity (resistances will be odd, beeps on true copper only).

1. Ground-symbol pin to: 9602 pin 8, 4004 pin 12, 4004 pin 5. Record which beeps.
2. +5V pin to: 9602 pin 16, 4004 pin 5, 4004 pin 12.
3. 4004 pin 12 (VDD) to every edge pin, COM and S.PASS included, and to every pad of the top-right glass diode and the large axial cap.
4. Confirm 4001 and 4002 VDD and VSS land on the same nets as the 4004's.
5. Result: a table of nets (0 V, +5, VDD) and where each enters the board. Answers whether ground is TTL 0 V or VDD.

## 3. Identify unknown parts with the meter

**Transistors with unknown markings (Q1 to Q10).** Out of circuit is ideal but desoldering is not on the table, so in circuit with care, and confirm anything odd by comparing two identical positions.

- Diode-test mode. A bipolar transistor is two diodes sharing the base. Find the pin that conducts (about 0.5 to 0.7 V) to both other pins in one polarity: that is the base.
- Red probe on base conducting to both: **NPN**. Black probe on base conducting to both: **PNP**.
- Of the two junctions, base-emitter usually reads a few millivolts higher than base-collector. That separates emitter from collector.
- If the meter has an hFE socket and a part ever does come out, it confirms type and gain.
- No diode behaviour, or conduction one way only between two pins: likely a JFET or MOSFET. Note it, do not guess.
- Record type, pinout and the marking (house numbers like HP 1854-xxxx are NPN and 1853-xxxx PNP in HP's numbering; look for the marking under magnification).

**Resistors:** read colour bands, then confirm in circuit with ohms; in circuit readings come out low when other paths are in parallel. Note both.

**Capacitors:** read the markings (the dipped tantalums, one partly reads "20 ... 15"). In circuit, ohms only to check for a short. No capacitance readings in circuit.

**Diodes:** diode test both ways. A zener reads as a normal diode forward; its zener voltage cannot be read with a meter, only with a current-limited supply later, out of the pre-power phase.

**The 6.8 axial (grey) and the dipped tantalums:** check each for a short across its leads. Tantalums of this age fail short. Any short is a replacement before power.

## 4. Trace the signals

From each 4004 pin outward, then from each test point inward. Continuity mode, one net at a time, highlight on a printout of the photo as you go. Order:

1. Clock: φ1 and φ2 test points back through the transistor drivers to the 9602 outputs. Note the timing R and C on each 9602 (pins for external R and C from the 9602 datasheet).
2. RESET: button, any RC, transistor, to 4004 pin 9 and the 4001/4002 reset pins.
3. TEST: U4-10 test point and its driver.
4. Bus: DATA 0 to 3 to 4004, 4001, 4002 (expect direct copper, no parts).
5. SYNC, CM-ROM, CM-RAM 0 to 3 to their test points; note any series resistors or buffers.
6. S.PASS and COM: what they connect to. That answers what S.PASS is.

## 5. Draw it

Redraw as a schematic, KiCad through Konnect (or on paper first). Commit the drawing and the parts table here. Send to the boards' owner for a second pair of eyes; they can check the logic while the meter work continues.

## Exit criteria before any power

- Every supply net known and its entry point identified.
- Every transistor typed and pinned.
- Every capacitor checked for shorts; failed ones replaced.
- Clock circuit drawn with its R and C values, with a predicted period and pulse width to compare against the scope later.
- Schematic reviewed with the boards' owner.
