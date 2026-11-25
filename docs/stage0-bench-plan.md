# Stage 0 bench plan: prove the board with an empty socket

Purpose: before a 4004 touches this board, show with a multimeter that every rail, every clamp and every level-shifting stage does exactly what the datasheets say, under load, at the limits, and when things go wrong. Every test has a pass criterion. Write the measured numbers down in the table at the end. A test that is "probably fine" has not passed.

Equipment: bench supply with adjustable current limit, multimeter (DC volts, DC current to 200 mA, ohms), the ten 1k 1/2 W resistors from the order (dummy loads, 15 mA each at 15 V), a 10k potentiometer, a USB 5 V source, wrist strap and mat.

Abbreviations: VSS rail means the +15 V rail. GND is the single board ground. "Pin n" means socket pin n with the socket empty.

Safety for the person: 15 V is harmless. Safety for the parts: the 100 µF capacitor holds charge, discharge through 1k before rewiring. Safety for the future chip: the whole document.

## Part A: the rails

### A1. 5 V input

Wire: USB 5 V, polyfuse MF-R025, 100 µF to ground. Nothing else yet.

- Measure 5 V rail to GND. Pass: 4.75 to 5.25 V.
- Measure current drawn (meter in series with the USB positive). Pass: under 1 mA (just the capacitor leakage).

### A2. TMR 1-0513 at no load

Add the module: pin 1 to GND, pin 2 to 5 V, pin 6 (-Vout) to GND, pin 4 (+Vout) is the VSS rail. 1 µF from VSS rail to GND.

- Before tying pin 6 to GND: with the module unpowered, measure resistance from pin 6 to pin 1. Pass: open circuit (isolation confirmed, no accidental bridge).
- Tie pin 6 to GND. Power on. Measure VSS rail to GND. Pass: 14.85 to 15.15 V (the datasheet's 1 % set accuracy).
- Measure input current on the 5 V side. Pass: 30 to 50 mA (datasheet 40 mA typical at no load).
- Record the exact no-load voltage. Call it V0.

### A3. Load steps

Add 1k resistors from VSS rail to GND one at a time: 1, 2, 3, 4 resistors are 15, 30, 45, 60 mA. 60 mA is 90 % of the module's 67 mA rating. Hold each step for a minute.

- Record VSS rail at each step.
- Pass: every reading within 75 mV of V0 (0.5 % load regulation from 10 to 90 % load) and all readings inside 14.25 to 15.75 V (the 4004's window).
- Record the 5 V side input current at 60 mA out. Expect roughly 60 mA x 15 V / (5 V x 0.79) plus 40 mA, so about 270 mA. Pass: under 350 mA. This is what the USB source must deliver.
- Touch the module after the minute at 60 mA. Warm is fine. Too hot to hold is a fail; stop and find out why.

### A4. Overload and recovery

Add a fifth and sixth 1k resistor: 75 mA and 90 mA. The datasheet current limit is 120 % minimum, 130 % typical of 67 mA, so 80 to 87 mA.

- At 75 mA: VSS rail may still be in regulation or may have started to fold back. Record.
- At 90 mA: expect foldback, a VSS rail well below 15 V. Record.
- Remove the extra resistors. Pass: VSS rail returns to within 75 mV of V0 without power cycling (automatic recovery).
- Note for later: the module's limit is above a healthy 4004's 40 mA. It protects the module, not the chip. First chip power-up uses the bench supply at a 60 mA limit for that reason.

### A5. Short circuit

With the module powered and no dummy load, short VSS rail to GND through the meter on its 200 mA current range for two seconds.

- Record the short-circuit current. Expect around 80 to 90 mA.
- Remove the short. Pass: VSS rail recovers to V0.

### A6. Zener clamp

Add the 1N4745A 16 V zener from VSS rail (cathode) to GND (anode).

- At 15 V the zener should draw only leakage. Measure VSS rail. Pass: unchanged from V0 within 10 mV.
- Prove the clamp works: disconnect the module's +Vout from the VSS rail and instead feed the rail from the bench supply through a 1k resistor. Raise the bench supply slowly from 15 to 18 V. Pass: VSS rail stops rising at about 16 V (15.2 to 16.8 V allowing for the zener's 5 % tolerance and its slope) while the bench supply goes on to 18 V.
- Put the module back. The clamp has now been seen doing its job. The 4004's absolute maximum is 20 V below VSS, so 16 V on a rail whose other end is ground is 4 V inside the limit.

### A7. Rail LED

Add the 5.6 V zener in series with an LED and a 1k resistor from VSS rail to GND.

- Pass: LED lit at 15 V, LED off when the bench supply (through the 1k from A6) is set to 6 V. This LED means "the 15 V rail is up", and it must be out before anyone touches the socket.

### A8. Cold start and brown-out

Feed the module's 5 V input from the bench supply instead of USB, current limit 500 mA. Meter on the VSS rail.

- Raise the bench supply from 0 V in 0.5 V steps. Record the input voltage at which the VSS rail jumps up. Pass: between 3.5 and 4.5 V (datasheet undervoltage lockout 4 V maximum), and the rail goes from nothing to about 15 V in one step rather than creeping up.
- Lower it again in 0.5 V steps. Record where the VSS rail drops out. Pass: it drops cleanly, no sustained half voltage.
- At 4.5 V input (the module's rated minimum) with 60 mA load: VSS rail within the 14.25 to 15.75 V window.

### A9. Soak

60 mA load, USB input, thirty minutes.

- VSS rail at start, at fifteen minutes, at thirty minutes. Pass: all within 75 mV of each other and inside the window.

### A10. The unregulated module, for comparison

Swap in the ROE-0515S (pins: check the [RECOM ROE series datasheet](https://recom-power.com/pdf/Econoline/ROE.pdf)). Repeat A2 and A3.

- Record the no-load and 60 mA voltages. Expect a spread of a volt or more. This is the RetroShield's rail. The number tells you how much margin the prior art was living on.
- Decision after this test: the ROE-0515S never powers a chip on this project. Put it back in the drawer.

### A11. D1 clamp current at the real rail

Added 2026-11-25 from the simulation (`docs/spice-lessons.md`, lesson 3). A 1N4745A at the low end of its tolerance band conducts milliamps at 15.0 V. Rail up, no chip, DMM in series with D1 on the mA range. Pass: under 100 uA. Milliamps: fit a 1N4746A (18 V) or leave the clamp out; record the number either way.

## Part B: the stages, socket empty

Restore the TMR 1-0513 as the VSS rail. Add the 74AHCT541 pair, the CD4504B, the BS170 stages, the four bus drivers, the comparators and the reference divider, the empty 16-pin socket, and the 12k pull-downs on pins 8, 11, 13, 14, 15, 16 and 1 to 4. Decoupling everywhere. Host connected, but for these tests the host just sets static levels on its pins (a few wires to 3.3 V or ground will do before the host firmware exists).

### B1. 74AHCT541 outputs

For each used channel: host pin at 3.3 V, then at 0 V.

- Pass: output at or above 4.5 V for a 3.3 V input and at or below 0.3 V for 0 V.
- Check that both output enable pins are at ground. A floating enable gives floating outputs.

### B2. CD4504B channels

For each of the six channels: input (via the 74AHCT541) at 5 V, then 0 V. Measure the CD4504B output.

- Pass: 14.5 V or higher for a 5 V input, 0.1 V or lower for 0 V. Six channels, twelve readings.
- Confirm SELECT (pin 13) is at 5 V (TTL input mode) and VCC (pin 1) reads 5 V, VDD (pin 16) reads 15 V.
- Measure VSS rail current with all six inputs static. Pass: under 1 mA from the CD4504B.

### B3. BS170 stages, TEST and RESET

For pin 9 and pin 10, each with its 10k pull-up to VSS rail.

- Gate (74AHCT541 output) at 5 V: pin reads 0.1 V or lower. The BS170 is sinking 1.5 mA through 10k; even a worst-case threshold part does this easily.
- Gate at 0 V: pin reads 14.9 V or higher.
- Pass on both pins, both states.

### B4. Bus drivers, three states each

For each of pins 1 to 4. Each has a 12k pull-down to ground, a 680 ohm series resistor to the driver node, a BS170 to ground and a TP0606 to VSS rail with its gate on a CD4504B output.

State 1, drive high: P-FET gate at 0 V (CD4504B input low), BS170 gate at 0 V.
- Pass: pin at 14.0 V or higher. The 680 ohm and the 12k form a divider: 15 x 12 / 12.68 is 14.2 V. The datasheet high limit for a 4004 input is 13.5 V. Record the actual value; the margin is about 0.7 V and it matters.

State 2, drive low: P-FET gate at 15 V, BS170 gate at 5 V.
- Pass: pin at 0.1 V or lower.

State 3, released: P-FET gate at 15 V, BS170 gate at 0 V.
- Pass: pin at 0.1 V or lower (the 12k pull-down holds it). Then lift the 12k and connect a 100k from pin to VSS rail instead: pin should now read close to 15 V. That proves nothing on the board is holding the pin. Put the 12k back.

Shoot-through check: in each state measure the VSS rail current and compare with the B2 figure. In state 1 expect about 1.2 mA per driven-high line (15 V through 680 plus 12k). Anything more than a few milliamps above that means a P-FET and a BS170 are on together. Fix before continuing.

Forbidden state: P-FET on and BS170 on at once puts 15 V across 680 ohms, 22 mA, with the current split between the two transistors. Nothing dies, but it must never happen with a chip in the socket, because the chip may also be driving. Make the host incapable of it (one signal, inverted, feeding both, with the P-FET path delayed a few ticks on turn-on).

### B5. Comparator reference

- Measure the reference node. Pass: 11.0 to 12.0 V (10k and 33k from 15 V gives 11.5 V; 5 % resistors move it by about 0.4 V).
- Measure it again with all nine comparators connected. Pass: unchanged (the inputs draw nanoamps).

### B6. Comparator thresholds

Wire the 10k potentiometer across the VSS rail and feed its wiper to each output pin in turn (pins 8, 11, 13 to 16, and 1 to 4). The 12k pull-down stays. Meter on the comparator output (the node pulled up to 3.3 V).

The signal goes to the non-inverting input and the reference to the inverting input, so the comparator output is high when the pin is above 11.5 V. For each of the nine comparators:
- Wiper at 0 V: output 0.4 V or lower.
- Wiper at 8.5 V (the worst-case 4004 low): output 0.4 V or lower.
- Wiper at 14.5 V (the worst-case 4004 high): output 3.0 V or higher.
- Sweep slowly and record the voltage at which the output flips. Pass: within 0.3 V of the reference node voltage measured in B5.
- Record the output low voltage with the 4.7k pull-up. Pass: 0.4 V or lower.

Nine comparators, five readings each.

### B7. Clock path, static

Host φ1 and φ2 outputs set to the idle state (host 1, pin high).
- Pin 6 and pin 7: 14.5 V or higher.
Host outputs set to the active state.
- Pin 6 and pin 7: 0.1 V or lower.

### B8. Clock path, running

Start the host clock generator at its real rate. Loop pin 6 and pin 7 back to the host through two spare comparator channels (there are 12 comparators in three LM2901 packages and 9 are used).

- The host measures, in its own ticks, the period of what comes back and the width of each pulse, and reports them. Pass: period 1.35 to 2.0 µs, widths 380 to 480 ns plus the comparator delay, which will add a few hundred nanoseconds to each edge equally and so leaves the period and roughly the width unchanged. If the host sees the right period and no overlap of φ1 and φ2, the clock generator is correct. Edge speed cannot be measured this way; that is scope work and it stays on the list.
- Meter on pin 6 while running: expect a reading well below 15 V (a meter averages the pulse). If it reads a steady 15 V or 0 V the clock is not getting through.
- VSS rail current while clocking into the empty socket. Pass: within 2 mA of the static figure (50 pF charged to 15 V 670 000 times a second is about half a milliamp per clock).

### B9. Full loopback

Host writes a nibble on the bus (state 1 or 2 on each of pins 1 to 4), reads the four comparator outputs back, and compares. All sixteen values. Then releases the bus (state 3) and reads back; all four must read as low pins.

- Pass: sixteen of sixteen match, and the released read is 0000 at the pins. Put an LED on the host that lights only when the loopback matches; it is the Stage 0 "done" lamp.

### B10. The pin table, the gate to Stage 1

Host in its Stage 1 starting state: clocks running, RESET asserted, TEST released, bus released. Rails up. Measure every socket pin to GND and fill in the table. Then switch the rails off and measure every pin again after ten seconds; everything must be at 0 V. Then switch on again and repeat the first column, because the second power-up is the one that catches sequencing problems.

| Pin | Name | Expected, rails up | Measured, first up | Measured, rails off | Measured, second up |
|---|---|---|---|---|---|
| 1 | D0 | 0.0 to 0.1 V (released, 12k down) | | 0 | |
| 2 | D1 | 0.0 to 0.1 V | | 0 | |
| 3 | D2 | 0.0 to 0.1 V | | 0 | |
| 4 | D3 | 0.0 to 0.1 V | | 0 | |
| 5 | VSS | 14.85 to 15.15 V | | 0 | |
| 6 | φ1 | averaged pulse, roughly 9 to 12 V | | 0 | |
| 7 | φ2 | averaged pulse, roughly 9 to 12 V | | 0 | |
| 8 | SYNC | 0.0 to 0.1 V (12k down, nothing driving) | | 0 | |
| 9 | RESET | 0.0 to 0.1 V (asserted) | | 0 | |
| 10 | TEST | 14.9 V or higher (released) | | 0 | |
| 11 | CM-ROM | 0.0 to 0.1 V | | 0 | |
| 12 | VDD | 0.00 V, and under 0.5 ohm to GND with power off | | 0 | |
| 13 | CM-RAM3 | 0.0 to 0.1 V | | 0 | |
| 14 | CM-RAM2 | 0.0 to 0.1 V | | 0 | |
| 15 | CM-RAM1 | 0.0 to 0.1 V | | 0 | |
| 16 | CM-RAM0 | 0.0 to 0.1 V | | 0 | |

Hard limits on every pin, every column: never above 15.5 V, never below -0.3 V. One reading outside is a fail for the whole board, whatever else passed.

Also before insertion:
- Resistance from pin 5 to pin 12 with power off: should read as the 12k pull-downs and the zener in parallel, a few kilohms, and never near zero.
- Resistance from every signal pin to pin 5 and to pin 12 with power off: no shorts.
- Orientation marked on the breadboard next to the socket: pin 1 and the notch.

## Part D: data-line learning sequence, signal generator, no 4004

Purpose: learn how one data line behaves in both directions before the full nibble exists. The circuit is the host interface v2 design (reviewed in `reviews/2026-11-23-host-interface-pdf-v2.md`). Two sources stand in for the two ends:

- **Host side, 5 V logic.** A signal generator set to a 0 to 5 V square wave (5 Vpp, +2.5 V offset, or its TTL/sync output) into the AHCT541 inputs, in place of D_OUT and BUS_OE. The AHCT541 runs from 5 V and accepts 5 V inputs. Static levels from jumpers to 0 V or 5 V for the truth-table steps.
- **"4004" side, 0 to 15 V.** A signal generator with DC offset driving the pin node, in place of the chip's output, through a series resistor that stands in for the 4004's output resistance.

Before connecting the generator to the pin node:

- **Check the generator's output range.** Most bench generators are limited to ±10 V into high impedance (offset plus half the amplitude). A 0 to 15 V swing needs a peak of +15 V; a worst-case 4004 swing of 8.5 to 14.5 V (6 Vpp, +11.5 V offset) needs +14.5 V. If the generator cannot reach the top, it cannot test the receiver threshold; note it and use step D4's fallback.
- **Set the output to high-impedance mode** so the displayed levels are the real levels.
- **Set it on the scope first, unconnected.** Confirm minimum and maximum before it touches the board. Measured 2026-11-23 on the bench DDS at 550 kHz, 0 to 15 V setting: plateaus correct but edge spikes to 18.0 V and −3.2 V with the long ground clip, 1.6 V with the ground spring. With 1 kΩ in series it gave a clean 14 V high and no overshoot. Use 1 kΩ in series as the standard "4004" source resistance for D4; it sits between the data-line ROH (150 to 250 Ω) and the CM-RAM ROH (1.8 kΩ). Never below 0 V (the BS170 body diode clamps through its 680 Ω) and never above 15.0 V (the TP0606 body diode back-feeds the 15 V rail through its 680 Ω).
- **Common ground**: generator ground to board ground at one point.
- **Pull-ups with no host attached:** the comparator pull-ups and the 74HC14 normally run from the host's 3.3 V. For bench work without a host, feed them from a 3.3 V bench supply. If 5 V is used instead, move them back to 3.3 V before any host is connected.

### D1. One data line, static

Wire one line: 15 V rail, one CD4504B channel, one TP0606, one BS170, a 680 Ω per FET, the 12k from the pin node to ground, one LS00 package doing the four-gate logic, one AHCT541 input and one output channel. Jumpers on BUS_OE and D. Meter on the pin, on the TP0606 gate, and in series with the 15 V supply.

| BUS_OE | D | Expect at pin | TP0606 gate | 15 V current change |
|---|---|---|---|---|
| 0 | 0 | 0 V (released, 12k) | 15 V | none |
| 0 | 1 | 0 V (released) | 15 V | none |
| 1 | 0 | 14.2 V (680 into 12k) | 0 V | +1.2 mA |
| 1 | 1 | 0 V | 15 V | none |

Pass: all four rows, and the supply current never shows the 11 to 22 mA that would mean both FETs on.

### D2. Receiver on that line, static

Add one LM2901 channel with the 3.0k / 10k reference divider, the 10k series resistor, 1M hysteresis and 2.2k pull-up. Bus released (BUS_OE = 0). Pot across 15 V, wiper through 1k to the pin node.

- Sweep up and down slowly. Record the pin voltage at which the comparator output flips each way. Pass: both flips within 0.3 V of the measured reference (about 11.5 V), about 30 mV apart.
- Pin at 8.5 V (worst-case 4004 low): output high (host reads 1). Pin at 14.5 V (worst-case high): output low (host reads 0).

### D3. Host-side square wave through the driver

Generator on D at 0 to 5 V, 100 kHz to 500 kHz, BUS_OE jumpered to 5 V. Scope four channels: D at the AHCT541 input, CD4504B output (TP0606 gate), the pin node, the comparator output.

- Pin follows D inverted: D high gives 0 V, D low gives 14.2 V.
- Measure the TP0606 turn-off: from the CD4504B output starting to rise to the pin starting to fall. Expected 150 to 600 ns. This is the number the release-delay parameter depends on.
- Measure the BS170 turn-on: fast, tens of ns. The difference between the two is the overlap; with a current probe or a 1 Ω sense resistor in the 15 V lead, see the overlap pulse, limited to about 11 mA by the two 680s.
- Measure the comparator delay from the pin crossing 11.5 V to its output crossing 1.65 V, both directions. Expected a few hundred ns; this feeds the 250 ns sampling margin.
- Then square wave on BUS_OE with D held at 0: watch the release. Time from BUS_OE falling to the pin falling below 13.5 V is the hold the 4004 would see.

### D4. "4004" side square wave into the receiver

Preferred source: the NE555 + CD4029B pattern generator (`pattern-generator-555-cd4029.md`), four lines counting at true 0 / 15 V with no overshoot, one per pin node through 1k each. The DDS generator below is the alternative.

Bus released (BUS_OE = 0). Generator with offset on the pin node through 1 kΩ in series (the overshoot-free setting measured above), then 1.8k for a CM-RAM line (ROH 1.8k max). Re-check Max and Min on the scope at the pin node with the ground spring each time the series resistor changes.

- Settings, in order: 0 to 15 V (7.5 V offset, 15 Vpp) if the generator reaches it; then the worst case, 8.5 to 14.5 V (11.5 V offset, 6 Vpp).
- Scope the pin and the comparator output. Pass: output toggles cleanly on every edge, no chatter at the crossing, delay as in D3.
- With 1.8k series and the 47k pull-down, the pin high reads about 14.4 V. Confirm it still reads as a 4004 logic 0.
- **Fallback if the generator tops out at +10 V:** drive a BS170 from the generator at 0 to 5 V, drain to the pin node through 220 Ω, with a 1k pull-up from the pin node to 15 V. The pin then swings about 0 to 15 V, inverted, at full edge speed. It stands in for the 4004 output well enough to test the receiver.

### D5. Contention, deliberately and briefly

Only after D3 and D4 pass. BUS_OE = 1, D = 1 (driver holding the pin at 0 V), generator through 220 Ω trying to drive 15 V. Pin current through the 680 limits it to about 20 mA. Confirm with the meter in the 15 V lead, then stop. This is what a turnaround collision looks like; recognise it on the scope.

### D6. Full nibble, then RESET and TEST

Build D0 to D3 and repeat D1 and D3 on each line. Then RESET and TEST per the v2 design: RUN and TEST jumpers, measure pins 9 and 10 at 0 V and 15 V, press the reset button and confirm pin 9 goes to 0 V regardless of RUN.

## Part C: the record

| Test | Measured | Pass |
|---|---|---|
| A1 5 V rail | | |
| A2 V0 no load, input current | | |
| A3 15 / 30 / 45 / 60 mA | | |
| A3 input current at 60 mA | | |
| A4 75 / 90 mA, recovery | | |
| A5 short-circuit current, recovery | | |
| A6 zener at 15 V, clamp voltage at 18 V in | | |
| A7 LED on at 15 V, off at 6 V | | |
| A8 UVLO up, UVLO down, 4.5 V in at 60 mA | | |
| A9 soak 0 / 15 / 30 min | | |
| A10 ROE no load, 60 mA | | |
| B1 AHCT541 high / low | | |
| B2 CD4504B six channels high / low, idle current | | |
| B3 TEST and RESET high / low | | |
| B4 bus x4, three states, shoot-through current | | |
| B5 reference | | |
| B6 nine thresholds, VOL | | |
| B7 clocks static | | |
| B8 clock period and width via loopback, running current | | |
| B9 loopback 16 of 16, release reads 0000 | | |
| B10 pin table, three columns, resistance checks | | |
| D1 truth table, one line, no both-on current | | |
| D2 receiver flip points up / down, 8.5 V and 14.5 V reads | | |
| D3 TP0606 turn-off ns, BS170 turn-on ns, overlap mA, comparator delay ns, release-to-13.5 V ns | | |
| D4 receiver at 0-15 V and 8.5-14.5 V, 1k and 1.8k series, pin Max/Min | | |
| D5 contention current | | |
| D6 four lines, RESET and TEST | | |

Date, module serial, host used, and who measured. Then, and only then, Stage 1 in `tutorial.md` section 6.
