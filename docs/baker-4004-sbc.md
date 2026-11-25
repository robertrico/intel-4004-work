# Scott Baker's 4004 / 4040 single board computer, read for this project

Sources: https://github.com/sbelectronics/4004, his write-up at https://www.smbaker.com/?p=3953, the Hackaday article of 2025-09-11, local clone at a local clone of github.com/sbelectronics/4004, directory `4004-smbaker`. Files: `schematic-normal-v0.23.pdf` (7 sheets), `schematic-zorkinator-bigram-v0.21.pdf`, `pld/bus/bus.pld` and `.jed`, `pld/fp/fp.pld` (an 8008 front-panel PLD, not for this board), firmware in `firmware/`, gerbers, `README.md`, `NOTES.md`. Read 2026-11-22. Prior art, not our design.

## What it is

A complete standalone MCS-40 computer built the way Intel intended, with modern conveniences. Loosely based on Jim Loos's 4004 SBC (the same Jim Loos whose 8008 SBC is in `8008-sbc.md`).

| Block | Part | Sheet |
|---|---|---|
| CPU | Intel 4004 or 4040 (two footprints, U$18 and U$2) | 1 |
| Clock | Intel P4201 or National INS4201 clock generator, crystal with 2 × 33 pF | 1 |
| Reset | 4201 RST_IN with 100 k to 5 V and 2.2 µF; push button; 4201 RST_OUT drives the MOS RESET net | 1 |
| Program memory | P4289 standard memory interface to a 28C256 / 27C256 / W27C512 / 39SF010 in a 32-pin socket with jumpers | 1, 3 |
| Page mapper | 74HCT670 4-bit register file, extends ROM from 4 KB to 16 KB in 1 KB pages | 3 |
| RAM | up to eight 4002 (two per CM-RAM bank, P0 pin selects chip 0 or 1) | 6 |
| I/O | P4265 programmable I/O on CM-RAM3, mode 12, 256 ports; 74HCT244 / 74HCT173 make them 8-bit | 5 |
| Expansion | RC2014-style bus (8-bit I/O only), one multimodule (SBX) socket, 74HCT138 chip selects | 5, 7 |
| Console | bit-bang serial through a 4002 output port and TEST input, or an 8251A UART with a 4.9152 MHz oscillator and 74HC4040 baud divider, MAX202 to RS-232 | 4 |
| Indicators | four LEDs on 4002 output ports through CD4049; four DIP switches read through a 74LS540 | 4 |
| Glue | one ATF22V10 (`bus.pld`): memory-map enable, RC2014 read/write strobes, high/low nibble latch control | 1 |
| Power | +5 V in (optional 7805), two isolated 2 W 5 V DC/DC modules in series make −10 V at 400 mA (he used 12 V-input modules because they were cheaper, with a jumper for the input type); 10 V zener and optional 2.2 k minimum load on the −10 V rail; +12 V TVS | 2 |

## Rails: Intel's, and why that matters for the calculator

VSS = +5 V, VDD = −10 V, TTL ground = 0 V. Every MCS-40 part on the board (4004/4040, 4201, 4289, 4265, 4002) sits on those rails.

The point that matters for our final machine: **the 4289 and 4265 are TTL-compatible on their memory and port sides only because their GND pin is at 0 V and their VSS at +5 V.** The 4289's address outputs (A0 to A11, RA lines), data inputs (OPR/OPA from the EPROM) and I/O lines, and the 4265's W/X/Y/Z port groups, swing between TTL ground and +5 V with nothing in between. The 4004 side of those chips runs on the MOS bus at +5 / −10 and never touches TTL. Likewise the 4201's crystal, reset input and RST_OUT.

In the +15 V / 0 V arrangement, those same TTL sides would sit at +15 V and +10 V. Nothing TTL could talk to them. So:

- **A standalone machine built from MCS-40 family parts (4289 or 4008/4009, 4265, 4002, 4201) runs on +5 / −10.** There is no choice in it.
- **The +15 / 0 arrangement is right for the bench harness**, where an FPGA reads the 4004 bus directly through comparators and no MCS-40 peripheral is on the bus.
- The 4004 does not care; it only sees 15 V between VSS and VDD. The same chip moves between the two boards.

the owner's HP exerciser has a +5 V test point and the Pro-Log board has +V / −V posts, consistent with both of them being +5 / −10 machines.

## How Baker crosses from MOS to TTL without level shifters

He does not shift the 4004 bus at all. The 4289 and 4265 do that job by design, and everything else crosses at the edges with a resistor and a clamp:

- **TTL into a MOS input (RESET, TEST).** 74HCT04 output, series Schottky (1N5819) to the MOS net, 10 k from the net to −10 V. TTL low pulls the net up to about 0.3 V (a MOS logic 0, since 0 V is near VSS); TTL high lets the diode block and the 10 k pulls the net to −10 V (MOS logic 1). The TEST input uses a 2N3906 PNP with 5.1 k to −10 V for the same purpose, driven from a CD4049. This is the Intel resistor-to-VDD trick with a diode added so the TTL side is never pulled negative.
- **MOS output into TTL (4002 output ports for LEDs and serial).** 4002 OUT pin with 10 k to −10 V, straight into a CD4049 input at 5 V. The 4049's input protection diode to ground clamps the negative excursion; the 10 k limits the clamp current to about 1 mA. Cheap, period-correct, and a known abuse of the protection diode that CD4049 datasheets tolerate.
- **RESET.** 4201 RST_OUT drives the MOS RESET net directly, with a 1N5819 from the 74HCT04 TTL reset so an external TTL reset can also assert it. Note on the sheet: the 0.1 µF decoupling at the 4201 is critical, "without it, reset line will ring like a bell".

## The 4201 clock generator

The chip that did the job our five-chip TTL board does. From the MCS-40 Advance Specifications, September 1974, pp. 40 to 47, now saved as `datasheets/MCS-40_Advance_Specifications_Sep74.pdf` (the November 1974 MCS-40 User's Manual is alongside it) [DS]:

- CMOS MSI, 16-pin DIP. VDD = VSS − 15 V ± 5 %. Pin 1 GND, pin 15 VSS (most positive), pin 4 VDD.
- Crystal between X1 (pin 7) and X2 (pin 8); "a simple series mode crystal-type circuit consisting of two inverters biased in the active region". Baker uses 33 pF to ground on each side.
- A 7-bit dynamic shift register circulates a single 1. **MODE pin 5** selects 7-bit or 4-bit length: MODE at VSS gives divide by 7; MODE at VDD gives the 4-bit mode, in which "the clock is divided by 2 and the relationship between the phases is equal; that is, φ1 pulse width, φ2 pulse width, φ1 to φ2 and φ2 to φ1 times are all equal". **That 4-bit mode is exactly the four-equal-tick Johnson design in `clock-generator.md`.** Intel built the same thing in 1974.
- Two output sets. MOS-level φ1 (pin 14) and φ2 (pin 3) "directly drive all MCS-4 clock inputs (up to 250 pF)" with "transition times of less than 50 ns". TTL-level φ1T (pin 2) and φ2T (pin 16), 2 mA sink, one standard TTL load.
- Reset: RESET IN (pin 12) with an RC between VDD and VSS, "a 1 MΩ / 0.5 µF combination will insure a 300 ms reset pulse"; RESET out (pin 13) "directly connects to all MCS-4 reset inputs". Baker uses 100 k and 2.2 µF with a button across the capacitor.
- Single-step logic for the 4040 (pins 6, 9, 10, 11), debounced on chip.

So with a 5.185 MHz crystal in 7-bit mode the period is 7 / 5.185 MHz = 1.35 µs, the datasheet minimum; in 4-bit mode a crystal near 4 MHz gives ÷2 then four equal states, 2.0 µs. Baker's notes mention 11.059 and 12.288 MHz crystals with "the 4201 board" and a `FAKE4201` firmware option whose only visible effect is one fewer NOP per serial bit; so the substitute runs the 4004 slightly faster than his real 4201 does. The substitute's circuit is not in the repo or the blog post; the classiccmp list records someone else's plug-in 4201 replacement built from three 4000-series CMOS parts, circuit not posted.

If a real 4201 turns up, it is the clock and the reset generator for the calculator. If not, our TTL clock timing does the same job; its CD4504 output stage is for 0 / 15 V only, and a +5 / −10 machine needs a MOS-level driver swinging +5 to −10 from TTL timing, which is the drive-side problem from `pmos-level-shifting-notes.md` section 2.6, still not designed.

## What the MCS-40 documents say about the interface chips

Advance Specifications [DS]:

- **4289**: "a single chip replacement for the 4008/4009 STANDARD MEMORY and I/O INTERFACE set ... enables the 4-bit microprocessor chip (4004 or 4040) to be used with standard memory components such as 1702 PROMs or 2102 RAMs." The Pro-Log's 4008 and 4009 are therefore the two-chip version of what Baker's board does with one.
- **4265**: "Input ports or lines are TTL compatible as well as the CLR/LD line. Outputs require external pull-up and +5 V, −10 V supply." And: "For TTL compatibility on the I/O lines, the supply voltage should be VDD = −10 V ± 5 %, VSS = +5 V ± 5 %, VDDI = GND. External pull-up is required for outputs."

That second sentence is the datasheet form of the point above: TTL compatibility on these parts is a property of the +5 / −10 / ground arrangement, stated by Intel, not a Baker convenience.

## Power budget, from his README

| Part | mA |
|---|---|
| 4004 | 30 to 40 |
| 4040 | 40 to 60 |
| 4201 | 20 |
| 4265 | 35 |
| 4289 | 30 to 40 |
| 4002 | 17 to 33 each |

RAM-less board 155 mA, two 4002 221 mA, six 4002 353 mA. All of that is on the −10 V rail through the DC/DC modules. Our 1 W TMR module at 67 mA powers a 4004 alone; a full MCS-40 machine wants several watts of −10 V.

## Firmware notes worth keeping

- First instruction at 0000 is a NOP, quoting Intel: "to avoid problems with power-on reset".
- `jcn t` jumps if TEST is at the positive voltage (+5 V), `jcn tn` if TEST is at −10 V. The firmware header credits Ryo Mukai's test4004 repository for the conventions.
- Bit-bang serial timing depends on the clock rate; his notes give working baud windows for two crystals.
- `bus.pld` is a clean example of decoding 4289 write strobes (`WOUT0 & !PM0`) into latch enables, and of the memory-map enable flip-flop built from PLD feedback.

## What this changes for us

1. **Two boards, two rail arrangements, and that is correct.** Bench harness at +15 / 0 with comparators into the FPGA. Standalone calculator at +5 / −10 with 4289 (or 4008/4009), 4002 and 4265, no level shifting on the bus because the MCS-40 interface chips are the level shifters.
2. **The calculator's parts list is Baker's parts list minus the expansion.** 4004, 4201 (or a clock substitute), 4289 or 4008/4009, one EPROM, two to four 4002, 4265 or 4001-style ports for the keypad and display, two DC/DC modules for −10 V, a 74HCT04, a CD4049, a few Schottkies. His gerbers and schematic are a working reference, MIT licensed.
3. **The clock board we are building is bench-only as drawn.** Its timing section (24.545 MHz, S163, LS163A, LS74, LS00) carries over to either machine. Its level stage (CD4504 at 0 / 15 V) does not carry to a +5 / −10 machine. For that machine: a real 4201, or a driver that swings +5 to −10 from TTL timing, which is the drive-side problem from `pmos-level-shifting-notes.md` section 2.6 again (P-channel FET from +5 with a pull-down to −10, or a push-pull pair). Not designed.
4. **Reset and TEST crossing** for the +5 / −10 machine: copy Baker's Schottky-and-10 k pattern. It is two parts per line and it is what the firmware expects.

## Questions for the boards' owner that this raises

- Did the HP exerciser or the Pro-Log board use a 4201, or a discrete clock? (The HP has 9602 one-shots, so discrete; the Pro-Log has a crystal can and Motorola CMOS near it, unknown. The 4201 was "available first quarter 1975" per the User's Manual, after both boards' date codes, so probably neither.)
- Does he have, or know a source for, a 4201, a 4289 or a 4265? The Pro-Log's 4008/4009 pair does the 4289's job if it works.
- Does he agree the final machine is +5 / −10 for the reasons above, with +15 / 0 reserved for the bench?
