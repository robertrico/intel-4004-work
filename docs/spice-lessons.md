# SPICE lessons (KiCad simulator)

Self-taught sequence. Tool: KiCad 10's built-in simulator (ngspice). LTspice was tried first on 2026-11-25 and dropped: the Mac build is the Windows program under CrossOver, menus render as "...", batch mode hangs on a Z: path, and the input state got stuck. KiCad is already the schematic tool here, so the simulator lives where the drawings live.

Scratch project: `hardware/spice-tests/` (root sheet holds lessons 1 and 2, sub-sheet `power` holds lesson 3) (not the host-interface project; that one has parts with no SPICE models and the simulator runs the whole hierarchy).

## How it works

- Inspect > Simulator opens the simulator window. It simulates the **whole project** from the root sheet, never one sheet.
- "New simulation tab" (first toolbar icon) picks the analysis. Transient (TRAN): time step and final time. Run is the play button.
- Probes: voltage probe icon then click a wire; current probe icon then click a part. Signals list on the right: Plot checkbox shows or hides a trace. Right-click a signal > Add measurement for max, min, average.
- Power symbols (`+5V`, `GND`) are **names only**. They supply nothing. Only `Simulation_SPICE` sources (VDC, VPULSE, VSIN, ISOURCE...) make volts. `GND` is the exception: it is SPICE node 0 and every circuit needs one.
- Net labels (`L`) name the traces: `V(/vc)` instead of `V(Net-_R1-Pad1_)`.
- `Device:R`, `Device:C`, `Device:L` simulate with no setup. Diodes, FETs, and anything exotic need a model: symbol Properties > Simulation Model tab.
- Source parameters appear in the Sim.Params field as short names. VPULSE: `y1` initial, `y2` pulsed, `td` delay, `tr` rise, `tf` fall, `tw` width, `per` period.
- Units: `k`, `Meg`, `u`, `n`, `p`, `m` (milli). `10M` is 10 milliohm. Mega is `Meg`.
- No `.step` in ngspice. Change the value, rerun. Sweeps from the command line use a shell loop.

## Lesson 1: 5 V and a resistor

`Simulation_SPICE:VDC` value 5, `Device:R` 1k, `power:GND`. TRAN 1u step, 1m stop. Probe: 5 V, 5 mA, 25 mW.

First attempt used a `+5V`-style power symbol instead of VDC and read 0 V everywhere: see "names only" above. Done 2026-11-25.

## Lesson 2: RC charge and VPULSE

VPULSE `y1=0 y2=5 td=0.1m tr=1u tf=1u tw=0.4m per=1m`, R in series, 1u to ground, label the cap node `vc`.

- R = 1k: tau = 1 ms, pulse is 0.4 ms, so vc ramps almost straight to 5 x (1 - e^-0.4) = 1.65 V (sim: 1.62 V), then decays.
- R = 100: tau = 0.1 ms, full charge by 4 tau = 0.4 ms, flat top, mirror-image discharge. P(V1) peaks at -250 mW = 5 V x 50 mA inrush at the edge.

That inrush is what the TMR 1-0513 sees charging the 47 uF on the 15 V rail. Done 2026-11-25.

## Lesson 3: the power sheet

Sheet 1 of the host interface rebuilt with simulation parts, in sub-sheet `power` of the scratch project.

Stand-ins: VDC 5 V, `R` 1 ohm (polyfuse MF-R025 cold), `C` 100u; the TMR 1-0513 as VDC 15 V behind `ROUT1` 1.25 ohm (0.5 % load regulation as ohms; no vendor model exists, and the 80..87 mA foldback, 110 mVp-p ripple and UVLO are not modelled, they are Stage 0 bench tests); `C` 1u, `C` 47u with a separate 0.5 ohm `R` for ESR (ngspice's ideal capacitor takes no `esr=` parameter); `D_Zener` D1 as the 1N4745A clamp, model `is=1n rs=2 bv=16 ibv=15.5m`; LED string `D_Zener` D2 `is=1n rs=1 bv=5.6 ibv=45m`, `LED` D3 `is=1e-18 n=1.8 rs=5`, `R` 1k; dummy load `R` 1k (one Stage 0 load, change it by hand for 2 to 4); `IPULSE` `y1=0 y2=40m td=2m tr=1u tf=1u tw=2m per=10m` as the 4004 starting to draw. TRAN 5u step, 5m stop.

Snags hit, in order, each read off the simulator log:
1. A resistor still valued `R`: "can't find model 'r'". Every part needs a number.
2. `esr=0.5` on a capacitor: "unknown parameter". Use a series resistor.
3. Diode parameters typed into Sim.Params with no device type: "could not find a valid modelname". Use Simulation Model tab, Built-in, Device Diode, so KiCad writes the `.model` line.
4. IPULSE parameters named `i1 i2`: KiCad's pulse names are `y1 y2 td tr tf tw per` for current sources too.
5. Rail at 9.44 V with 4.4 A through D1: the diode conducting forward although the drawing had the bar on the rail. Field **Sim.Pins** read `1=A 2=K` while the symbol's pin 1 is K. KiCad's default diode mapping had the pins backwards for `Device:D_Zener`. Fix: Sim.Pins `1=K 2=A` on every diode. Also `ibv=15.5` meant 15.5 amps; `15.5m`.
6. LED string at 12.7 mA with 0.44 V across D2: a forward drop, not a 5.6 V zener. D2 still had the pins swapped.

Results, all matching pencil: v15 14.972 V idle, 14.922 V during the 40 mA pulse (50 mV = 40 mA x 1.25 ohm), I(Rout) 22.6 to 62.6 mA, LED 7.67 mA, D2 drop 5.56 V, LED drop 1.74 V, D1 1 nA. Step corners rounded by tau = 1.25 ohm x 47 uF = 60 us.

Done 2026-11-25.

### Lesson 3 experiment: the clamp zener's tolerance

D1 breakdown voltage stepped by hand through the 1N4745A band (15.2 to 16.8 V at 15.5 mA) and below it. Rail 14.97 V, one dummy load, no chip. Unit trap on the way: `bv=15.2m` is 15.2 millivolts and silently does nothing; volts take no suffix.

| bv | D1 current | D1 power | Rout total |
|---|---|---|---|
| 16.0 | 1 nA | 0 | 22.6 mA |
| 15.2 | 2.3 uA | 34 uW | 22.6 mA |
| 15.0 | 3.4 mA | 51 mW | 26.0 mA |
| 14.9 | 20 mA | 300 mW | 42.6 mA |

Three tenths of a volt of zener tolerance spans "off" to "eating the supply". With the 4004's 40 mA on top of the last row, 83 mA, inside the TMR's 80 to 87 mA foldback band. SPICE's knee is a sharp exponential and a real zener's is softer, so the sim gives the shape of the risk, not the number. Decision pending one bench measurement: D1 current at 15.0 V in Stage 0 Part A. Tens of uA, keep it. Milliamps, swap to 1N4746A (18 V, still under the 4004's 20 V absolute maximum) or leave the clamp off and trust the regulated TMR.

Done 2026-11-25.

### Next

DC sweep: new simulation tab, DC, sweep I1 from 0 to 100m in 1m steps, plot v15 against load current; one run replaces a load-step series. Then the clock sheet.
