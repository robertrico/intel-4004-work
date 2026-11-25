# Running a real Intel 4004 from a 3.3 V host: step by step

This is the walkthrough for the project. It assumes the parts in `../bom/pmos-4004-bom.csv` are on the bench, a bench power supply with an adjustable current limit, a multimeter, breadboards, and a host that can generate two clock phases at a few hundred kilohertz (an ECP5 FPGA or a Pico 2 using PIO). The reasoning behind every number is in `pmos-level-shifting-notes.md`; this document only tells you what to do and in what order. Section references like "notes 2.7" point there.

The order of the steps is the safety plan. Nothing with a 4004 in it happens until everything without a 4004 in it has passed.

## 0. What may be arriving, and what not to do to it

Two boards were photographed by the donor. Everything below about them is read from the photographs and should be confirmed with the boards in hand.

**HP MCS-4 Exerciser, COS-2.** A white ceramic C4004 (date code F6015), a P4001-0009 mask ROM, two P4002-1 RAMs, three 9602 dual one-shots (the clock generator), a reset push button, and labelled test points along the edge: DATA 0 to 3, CM-RAM 0 to 3, CM-ROM, TEST, SYNC, CLOCK φ1 and φ2, +5V, S.PASS, COM. It is a complete, self-clocked MCS-4 system built to exercise a 4004. The 4004 appears to sit in a socket.

**Pro-Log PWM 100674XA.** A white ceramic C4004 (N0166), a C4008 and C4009 (the standard-memory interface pair, gold lids), four C1702A EPROMs in sockets with handwritten labels, two P4002-1 RAMs, 7404 and 7416 TTL, Motorola MC145xx CMOS near a crystal can, and three power posts marked +V, -V and GND. Also a complete system, with program memory in EPROM. The 4004 appears socketed.

Both boards run the chips on Intel's rails, +5 V and a negative supply, not on the +15 V and ground used by the board you are about to build. That is fine for the chip. It only ever sees the 15 V between VSS and VDD. It matters for what you connect to what.

Rules for the boards and the chips in them:

1. Do not power either original board until you know what rails it expects. The 4004 wants VSS minus VDD of 15 V within 5 %. A 1702A EPROM wants its own negative rails (VDD and VGG at about -9 V). Which post gets which voltage is not knowable from the photo. Ask the donor what they ran them from, or trace the board. Fifty-year-old electrolytic capacitors (the Pro-Log board has several, 1 µF and 47 µF at 25 V) also deserve a look before any voltage reaches them.
2. Do not desolder a ceramic 4004. If the chip is in a socket, it can come out; if it is soldered, it stays on that board and the board becomes the test fixture instead of this one.
3. Remove a socketed chip only with a DIP extractor or by levering a thin flat blade alternately under each end a millimetre at a time. Never from one end only. The gold-plated leads bend once and break the second time.
4. Keep every chip in conductive foam or in its socket. Handle by the ends of the package. Wrist strap on, mat under the board, every time.
5. No chip goes into the new board until Stage 0 (the whole of `stage0-bench-plan.md`) has passed. That is the entire point of Stage 0.
6. **Running the donated boards as built is a branch of the plan, not a fallback.** Per the donor (2026-11-22): both boards came out of HP surplus around 1980 and have not been powered since. The HP board was a sales demo for HP's early logic analyzers: a self-clocked 4004 system whose test points wiggle in a recurring sequence, with a 4001 that probably holds demo patterns. That is a bus-on-labelled-pins instrument, the thing the FPGA harness was going to imitate. The Pro-Log is the CPU card from an M900-series PROM programmer. If the donor confirms the rails and what they do, each gets the same discipline as the new board in a different order: inspect with power off; pull **every socketed MOS part** with the extractor, one careful lift each, into labelled foam with the socket position written down (HP: 4004, 4001, both 4002s; Pro-Log: 4004, 4008, 4009, both 4002s, all four 1702As), leaving the TTL and CMOS logic in place; power the empty board on the bench supply with a current limit; measure the rails at every empty socket pin, including the three supply pins of each 1702A socket, and confirm nothing sits above VSS; scope the clock and reset test points against the datasheet table in `clock-generator.md`; then repopulate one chip at a time with a power cycle between each, in the order the system needs them (HP: 4004, then 4001, then 4002s; Pro-Log: 4004, then 4008 and 4009, then one 1702A, then the rest, then 4002s). Soldered parts stay and are covered by the measurements around them. A board that runs is worth more than its chips, and it is the first machine that will ever show you SYNC.

## 1. What you are building

A 3.3 V host talks to the 4004 through three kinds of stage. Host to chip goes through a 74AHCT541 at 5 V and then either a BS170 open-drain transistor (TEST, RESET, and the low side of the bus) or a CD4504B level shifter (both clocks, and the gates of the P-channel FETs that make the high side of the bus). Chip to host goes through LM2901 comparators powered from 15 V with an 11.5 V threshold and open-collector outputs pulled up to 3.3 V, straight into the host.

Rails: 5 V from USB for the 74AHCT541 and the CD4504B input side, 15 V from the TMR 1-0513 module for the 4004, the CD4504B output side, the P-FET sources and the comparators, 3.3 V from the host for the comparator pull-ups only. All three grounds meet at one point.

The 4004 pinout, from the datasheet (the Intel 4004 datasheet, two scans, from [bitsavers](http://www.bitsavers.org/components/intel/MCS4/), page 8-16):

| Pin | Name | Direction | Stage on this board |
|---|---|---|---|
| 1 | D0 | bidirectional | BS170 low + TP0606 high via CD4504B; LM2901 read |
| 2 | D1 | bidirectional | same |
| 3 | D2 | bidirectional | same |
| 4 | D3 | bidirectional | same |
| 5 | VSS | power | +15 V |
| 6 | φ1 | in | CD4504B channel, direct |
| 7 | φ2 | in | CD4504B channel, direct |
| 8 | SYNC | out | LM2901 |
| 9 | RESET | in | BS170, 10k pull-up to +15 V |
| 10 | TEST | in | BS170, 10k pull-up to +15 V |
| 11 | CM-ROM | out | LM2901 |
| 12 | VDD | power | ground |
| 13 | CM-RAM3 | out | LM2901 |
| 14 | CM-RAM2 | out | LM2901 |
| 15 | CM-RAM1 | out | LM2901 |
| 16 | CM-RAM0 | out | LM2901 |

Every 4004 output also gets a 12k resistor to ground (Intel's "for TTL compatibility" pull-down, notes 2.4). Every bus line gets a 680 ohm resistor in series with its driver, the RetroShield's contention limit.

Logic polarity: the 4004 is PMOS with negative logic. A logic 1 on a pin is the voltage nearest VDD, which on this board is ground. A logic 0 is near VSS, +15 V. So "assert RESET" means pull pin 9 to ground with the BS170 on, and the comparators read a 1 when the pin is low. Invert in the host once, in one place, and never think about it again.

Per-stage circuits (values are from notes 2.6 and 2.7):

- **BS170 open drain** (TEST, RESET): gate from a 74AHCT541 output, source to ground, drain to the pin, 10k from the pin to +15 V. Gate high pulls the pin to ground.
- **CD4504B** (φ1, φ2, P-FET gates): VCC pin 1 to 5 V, VDD pin 16 to 15 V, VSS pin 8 to ground, SELECT pin 13 to 5 V (TTL-to-CMOS mode, input high threshold 2.0 V, so 74LS, 74HC, 74HCT or the 74AHCT541 can all drive it). With SELECT at ground the inputs want 3.5 V and a 74LS output's guaranteed 2.7 V high is not enough. Inputs on pins 3, 5, 7, 9, 11, 14; outputs on 2, 4, 6, 10, 12, 15. The datasheet states "VCC can exceed VDD" and "independence of power-supply sequence considerations", so 5 V rising before 15 V is allowed.
- **Bus driver** (D0 to D3): TP0606 source to +15 V, gate from a CD4504B output, drain through 680 ohms to the pin. BS170 drain on the pin side of the same 680 ohm resistor, source to ground, gate from a 74AHCT541 output. Host rule: never turn both on. A CD4504B output at 15 V turns the P-FET off; at 0 V it turns it on.
- **Comparator** (all nine outputs): LM2901 VCC to +15 V, GND to ground. Signal into the non-inverting input, 11.5 V reference into the inverting input (10k from +15 V to the reference node, 33k from the node to ground gives 11.5 V; one divider feeds all nine). Output through 4.7k to 3.3 V, and that node goes to the host pin. Output high means the 4004 pin is near VSS, which is a logic 0 in the 4004's own convention.
- **Decoupling**: 100 nF at every IC supply pin, 1 µF at the 4004's VSS pin to ground and across the module output, 100 µF on the 5 V input side of the module.
- **Host lines and power order.** The 4004 never has 15 V unless the host is powered, configured and driving every line to its idle value. That rule replaces any hardware default-state scheme. Order, every time: host powered and configured (RESET asserted, bus released, TEST inactive); 5 V on; 15 V on; hold RESET 64 clocks or more, then release from the host. Before any reflash, reconfiguration, unplug or host power cycle: 15 V off first. Power down: 15 V off, then 5 V, then the host. Two 74AHCT541s, non-inverting, carry every host line; tie both output enables (pins 1 and 19) to ground; 10 k pull-downs on the 541 inputs so an unplugged ribbon with 5 V up reads as all-idle. Nothing the host does or fails to do can take a 4004 pin outside 0 to 15 V; the only consequence of breaking the order is bus contention through the 680 Ω resistors and a chip that is not in reset, which is a debugging problem, not a damage mechanism. A manual reset for the bench is a push button from the RESET BS170 gate to 5 V through 1 k.

## 2. Parts you need beyond the order

From the drawer: 680 ohm (4), 10k (2 pull-ups plus the divider), 33k (1), 12k (9), 4.7k (9), 100 nF ceramics (about 10), a few LEDs with 330 ohm resistors for the host side, a 16-pin DIP socket for the breadboard phase (the Mill-Max sockets are for the final board; on a breadboard use any cheap one, or plug the Mill-Max into the breadboard and accept that its pins are tight), a 10k potentiometer for the comparator test, hookup wire.

Tools: bench supply with current limit, multimeter, wrist strap and mat, DIP extractor, magnifier.

## 3. Build order on the breadboard

Alongside this: the two donated boards get studied statically, power off, with the meter, until their power sections and clocks can be drawn from the boards themselves and every passive is accounted for (section 0, rule 6). The breadboard runs in parallel as a learning exercise: the host interface (host interface v2 design (`~/Downloads/4004Schematicv2.pdf`), reviewed in `reviews/`) built stage by stage with signals injected and measured, no 4004 attached. No chip goes into it until Stage 0 has passed, and the final custom board is designed on what both the breadboard and the donated boards teach.

Build in this order and test each piece as you add it (the tests are in the Stage 0 plan):

1. The 5 V input: USB, polyfuse, 100 µF, then the TMR 1-0513. Nothing else. Test the rail.
2. The 16 V zener and the rail LED (5.6 V zener in series with an LED and a resistor from +15 V, so the LED only lights above about 8 V).
3. The 74AHCT541 pair. Host pins in, 5 V out. Test.
4. The CD4504B. Test all six channels to 15 V.
5. The BS170 stages for TEST and RESET. Test.
6. The four bus drivers. Test each in all three states.
7. The nine comparators and the reference divider. Test with the potentiometer.
8. The empty 16-pin socket, wired to all of the above, with the 12k pull-downs on the output pins.
9. The full loopback and the pin-by-pin voltage table. This is the Stage 0 gate.

Keep the 15 V wiring on its own breadboard rail strip, marked, with nothing from the 5 V or 3.3 V world sharing it. Keep the clock wiring short.

## 4. Stage 0: the empty-socket proof

Follow `stage0-bench-plan.md`. Do not skip tests because earlier ones passed. The last item in it is the insertion checklist.

## 5. The host side

The host does three jobs before any program runs: generate the clocks, drive RESET, and watch SYNC.

**Clocks.** Build these in TTL first, then move them into the host once the bus logic exists. Full pin-level design in `clock-generator.md` (24.545 MHz can, 74S163 divide by 4 then 74LS163A modulo 3, 489 ns tick). The principle: a tick of about 489 ns clocks two flip-flops wired as a Johnson counter (FF A: D = /QB; FF B: D = QA) through states 00, 10, 11, 01; /φ1 = NAND(/QA, /QB) is low only in state 00 and /φ2 = NAND(QA, QB) only in state 11. Each phase one tick low, one tick between, four ticks per cycle, glitch-free because only one counter bit changes per tick. The tick length matters more than it looks: the datasheet measures pulse width at the 90 % level and the gaps at the 10 % level, so each reads one CD4504B edge time (40 to 80 ns) shorter than the tick. 489 ns passes every limit at the slowest edge the CD4504B datasheet allows; 400 to 444 ns designs do not. The tick must be retuned if the 15 V driver changes, because the width upper limit also moves with the edge time. Outputs go to the CD4504B inputs; with SELECT high any family drives them.

Two low-going pulses that never overlap, both resting high (at VSS, 15 V at the pin). From the datasheet (notes 2.4): period 1.35 to 2.0 µs, pulse width 380 to 480 ns, φ1 trailing edge to φ2 leading edge 400 to 550 ns, φ2 trailing edge to φ1 leading edge at least 150 ns. A 20 MHz tick (50 ns) and a 30-tick cycle gives:

```
ticks  0 to  7   φ1 active   400 ns
ticks  8 to 16   both idle   450 ns
ticks 17 to 24   φ2 active   400 ns
ticks 25 to 29   both idle   250 ns
```

Period 1.5 µs. On the FPGA this is a 5-bit counter and two compares. In PIO it is the same with the cycle count scaled to the PIO clock. The host outputs a 1 where the pin must be low; the 74AHCT541 and CD4504B are non-inverting, so a host 1 is a 15 V pin and a host 0 is a 0 V pin. Idle is therefore host 1 on both clocks and a pulse is a 0.

The clocks must run continuously whenever the 15 V rail is up with a chip in the socket. The 4004 is dynamic logic and the datasheet maximum period is 2.0 µs. Never stop them to "look at something". Single-step by feeding NOPs, not by halting the clock.

**RESET.** Assert it (BS170 on, pin at ground) before the clocks start, keep it asserted for at least 64 clock periods after the clocks are stable (the datasheet asks for 64 cycles, eight machine cycles, to clear everything), then release. Assert it again any time you are not sure what state the chip is in.

**SYNC.** The chip asserts SYNC for one clock period in every eight (the X3 state). Count SYNC edges in the host and light an LED when they arrive at one eighth of the clock rate. That is the first sign of life, and it appears with RESET still asserted. Nothing else is needed for Stage 1.

**Bus timing** for later stages (notes 2.4 and 2.5, and the RetroShield `t4004` firmware in [retroshield-teensy](https://github.com/8bitforce/retroshield-teensy) as the worked example): the instruction cycle is A1 A2 A3 M1 M2 X1 X2 X3, one clock period each, SYNC marking X3. The chip drives the address nibbles during A1 to A3 and the host must drive the instruction nibbles during M1 and M2, data valid 350 ns before the trailing edge of φ2 and held 40 ns after it. Host rule, in ticks from SYNC (which the host sees at the φ2 trailing edge of X3): command the bus drive at the φ1 leading edge of M1 (tick 0 of that period); command the release exactly at the φ2 trailing edge of M2, and likewise for X2 on I/O reads. The CD4504B's 140 to 280 ns delay then provides the 150 ns hold on lines driven high (the 12 k pull-down provides it on lines driven low), and the 449 ns tφD2 gap keeps the worst-case release about 130 ns clear of the CPU taking the bus at X1. Releasing early breaks the 40 ns tH; releasing late collides with the CPU. The host sees chip output through about 300 ns of comparator delay; sample late in the phase.

## 6. Stage 1: first power with a chip

Only after the Stage 0 gate.

1. Everything off. Discharge the 15 V rail through a 1k resistor and confirm 0 V on the meter.
2. Replace the TMR 1-0513 with the bench supply for this first run: bench supply set to 15.00 V, output off. Current limit: the empty-socket draw measured in Stage 0 (the whole interface shares this rail, roughly 15 to 20 mA) plus 50 mA, not a flat 60 mA. The 50 covers the 4004's own IDD (40 max) plus what it drives into the pull-downs and its clock pins (about 7). Bench supply positive to the VSS rail, negative to board ground. The module comes back later once the chip is known good; the module's own current limit sits at 80 to 87 mA, which is above what a damaged chip could draw.
3. Host running, clocks running into the empty socket, RESET asserted. Verify on the meter: pin 6 and pin 7 read about 15 V (clocks rest high; a meter on a 30 % duty pulse reads lower, about 10 V, which is also fine), pin 9 reads 0 V, pin 10 reads 15 V, pin 5 reads 15 V, pin 12 reads 0 V.
4. Host stopped, 5 V off, 15 V off. Wrist strap. Insert the chip. Pin 1 is at the notch end; on this socket orient it the same way as every other DIP. Check twice. Check that no lead folded under.
5. 5 V on. Host on, clocks running, RESET asserted. Bench supply output on. Watch the current reading: 30 to 40 mA is right. Anything above 55 mA, off immediately.
6. SYNC LED. If it is lit, the chip is executing its idle sequence. Leave it for a few minutes and watch the current stay steady.
7. Off in the reverse order: 15 V off, then host, then 5 V.

If current is right but SYNC never appears: check pin 8 with the meter (expect a brief pulse, so a reading somewhat below 15 V), check the comparator on that channel with the potentiometer again, check that the clocks are actually pulsing (loop them back through two spare comparators as in Stage 0).

## 7. Stage 2: free run

Release RESET. During A1, A2, A3 the chip puts the low, middle and high nibble of the program counter on the bus. The host reads them through the comparators (remember the polarity) and, during M1 and M2, drives 0000 and 0000, which is NOP. The program counter walks from 0 to 4095 and wraps. Display the address on the host. Seeing it count is proof that the fetch and execute machinery works and that the bus works in both directions.

## 8. Stage 3: a program

Put a small table in the host's memory and return bytes by address instead of always NOP. The RetroShield firmware shows how the host decides which cycle it is in by counting from SYNC. A program of a few instructions can load a value and write it to a RAM output port (WMP) in a loop; the host sees the write at X2 with CM-RAM asserted and toggles an LED. The LED is then blinking under 4004 software control. That is the goal of this parts list.

## 9. Stage 4 and beyond

Emulate the 4001 ROMs and 4002 RAMs fully and the Busicom calculator firmware runs; the RetroShield `t4004` sketch is the reference for the whole protocol. Once that works, the HP exerciser's P4001-0009 ROM can be read out: either log the bus while the exerciser runs its own program, or move the 4001 onto the bench board and let the host walk all 256 addresses with the 4004 fetching. the donor expects it holds logic-analyzer demo patterns rather than a program; a dump settles it and is worth publishing either way. The Pro-Log board's 1702A EPROMs are easier, any modern programmer reads a 1702A, and they hold the firmware of a Pro-Log M900-series PROM programmer, which is what that board came out of.

## 10. From breadboard to PCB

When the breadboard version has run a program, lay it out. What changes:

- A ground plane. Clock edges of tens of nanoseconds on a breadboard are a compromise; on a PCB with the CD4504B next to the socket they are what the datasheet says.
- The Mill-Max machined socket for the chip. Nothing else touches its leads.
- The 15 V section in its own area, zener and 1 µF at the socket.
- A header for the host carrying the 17 signals (two clocks, RESET, TEST, four bus-out, four bus-in, five status) plus 3.3 V and ground. Keep it host-agnostic: FPGA board or Pico 2 plugs into the same header.
- Test points on every socket pin so the Stage 0 table can be repeated on the finished board before the chip goes in. That table is run again on every new board, every time.

Keep the design files in this repo. Nothing of this belongs in the RetroShield repositories.
