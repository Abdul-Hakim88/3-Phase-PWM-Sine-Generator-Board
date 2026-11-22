# 3-Phase PWM Sine Generator Board (Arduino Nano + LM386)

A low-power three-phase sine source. An Arduino Nano generates three sine-modulated PWM signals 120 degrees apart, an RC filter turns each one into a clean sine, and an LM386 amplifier per phase gives it enough current to drive a small load.

This repository contains the **hardware**: the circuit, the PCB and the simulation. The firmware is not mine. It is the open-source sketch from [cmasenas/3-Phase-Sine-Arduino](https://github.com/cmasenas/3-Phase-Sine-Arduino), see [Firmware](#firmware).

<img width="1280" height="960" alt="Testing With NANO" src="https://github.com/user-attachments/assets/9e1c6b6e-d1c5-41d0-b277-316bc0d4d228" />


---

## Features

- Three outputs, 120 degrees apart, from Arduino pins 9, 10 and 11
- 31.37 kHz PWM carrier, software-set output frequency (DDS)
- First-order RC filter per phase (1 kohm + 100 nF)
- One LM386 (gain 20) per phase, powered from a single regulated 12 V supply
- No output coupling capacitors: outputs rest at about half the supply and are meant to be used between phases (see [Output stage](#output-stage-and-the-load))
- Simulated in Proteus (project files included)

---

## How it works

```
Arduino pin --> RC filter --> 10uF --> divider --> LM386 --> Zobel network --> phase output
 (PWM)          1k+100nF     (blocks   10k / 1k    gain 20
                             DC)
```

1. **PWM sine (firmware).** The sketch runs Timers 1 and 2 in phase-correct PWM with prescaler 1, so the carrier is 16 MHz / 510 = 31.37 kHz. An interrupt on every PWM cycle steps a 32-bit phase accumulator, and a 256-point sine table sets the duty cycle. The other two phases read the table at offsets of 85 and 170 entries (119.5 and 239 degrees).
2. **RC filter.** 1 kohm + 100 nF per pin (corner about 1.6 kHz) removes most of the 31 kHz carrier and leaves the 20-25 Hz sine, about +-2.4 V on a 2.5 V DC offset.
3. **Coupling capacitor.** 10 uF removes the DC offset, so the LM386 sees a signal centred on 0 V.
4. **Divider.** 10k / 1k (ratio 1/11) reduces the signal to about +-0.22 V. The LM386's fixed gain of 20 brings it back to about +-4 V without clipping, and the input stays inside the chip's +-0.4 V limit.
5. **LM386.** Supplies the current the Arduino pin cannot (its limit is about 20 mA). The output rests at half the supply, about 6 V.
6. **Zobel network.** 10 ohm + 47 nF from the output to GND keeps the LM386 stable with reactive loads.

### Frequency math

```
f_out = 31372.5 * delta / 2^32
delta = 2^24 * f / 122.549        (122.549 = 31372.5 / 256)
```

---

## Output stage and the load

- Each LM386 output rests at about 6 V DC with the sine on top. The board has **no output capacitors**, so the DC level is not blocked.
- All three chips share the same supply, so all three outputs rest at the same DC level. A load connected **between** the phases (for example a star with a floating neutral) therefore sees only the AC sine and no DC.
- **Do not connect a load from an output to GND**, and do not ground the star point. That puts about 6 V DC across the load and wastes power in the chip.
- All three phases must be connected and powered together. A single phase alone has no return path.
- Keep the load resistance above about 50 ohm per phase. The LM386 is not meant for low-impedance loads, and an output short is not something to rely on its protection for.
- Measuring: every output sits at about 6 V DC, so use AC coupling on the scope, or measure between two outputs (channel A minus channel B).
- 
<img width="1919" height="1028" alt="Simulation Output" src="https://github.com/user-attachments/assets/52f9ccf8-900c-4212-8fe2-96a340e68114" />

---

## Expected results

Simulation, 12 V supply, 10k / 1k divider:

| Quantity (per phase) | Value |
|---|---|
| Output amplitude | about 8 V peak-to-peak (about +-4 V, 2.9 V RMS) |
| Line-to-line voltage | about 1.73 x phase voltage |
| Phase spacing | 1/3 of a period (16.7 ms at 20 Hz, 13.3 ms at 25 Hz) |
| Output DC level (LM386 pin 5) | about 6 V |

The real LM386 may swing slightly less than the Proteus model. If the sine shows flat tops on a scope, increase the 10k series resistor of the divider to 15k or 22k.

---

## Bill of materials

**Per phase (x3)**

| Part | Value | Notes |
|---|---|---|
| U | LM386N (DIP-8) | The 4-12 V variant; do not exceed 12 V |
| R filter | 1 kohm | PWM filter |
| C filter | 100 nF | PWM filter |
| C coupling | 10 uF, 16 V+ | + side toward the filter |
| R1 | 10 kohm | Divider, series |
| R2 | 1 kohm | Divider, to GND |
| C bypass | 10 uF, 16 V+ | LM386 pin 7 to GND, + on pin 7 |
| C decoupling | 100 uF, 25 V + 100 nF | LM386 pin 6 to GND, close to the chip |
| Zobel | 10 ohm + 47 nF, 16 V+ | LM386 pin 5 through 10 ohm and 47 nF to GND |

<img width="1280" height="960" alt="Altium To Reality" src="https://github.com/user-attachments/assets/a2f665b2-b3e2-4398-8dc1-2fec9d052922" />


**Shared**

- Arduino Nano (ATmega328P)
- **Regulated** 12 V DC supply (the Nano is powered separately, for example by USB)
- Common GND between the Nano, the supply and all three LM386 chips

---

## LM386 connections (per phase)

| Pin | Connection |
|---|---|
| 1 | Open (gain 20) |
| 2 | GND |
| 3 | Divider output (R1 / R2 junction) |
| 4 | GND |
| 5 | Phase output (with the Zobel network to GND) |
| 6 | +12 V, with 100 uF + 100 nF to GND |
| 7 | 10 uF to GND |
| 8 | Open (gain 20) |

Signal path before pin 3: `Arduino pin -> 1k -> node A (100 nF to GND) -> 10 uF -> 10k -> pin 3 (1k to GND)`.

---

## Firmware

The firmware is **not part of this project's own work**. It is the sketch `DDS_Generator.ino` from [cmasenas/3-Phase-Sine-Arduino](https://github.com/cmasenas/3-Phase-Sine-Arduino) by C. Masenas (3-phase PWM sine, 2016), which is itself modified from DDS code by KHM 2009 / Martin Nawrath. Download it from that repository.

Things to know when using it with this board:

- Board: Arduino Nano (ATmega328P), 16 MHz.
- Outputs: pin 11 (Timer2), pin 10 and pin 9 (Timer1).
- The frequency is set with `changeFreq(f)` in the sketch. The example `loop()` alternates between 20 Hz and 25 Hz every 10 seconds.
- The amplitude does not change with frequency (no V/f control), and the sine gets coarser as the frequency rises. 20-50 Hz is comfortable.
- Pin 7 toggles inside the interrupt, so you can measure its run time on a scope. Pin 6 is set high as an enable line, which this board does not use.
- A comment in the sketch says "fast PWM", but the register bits select phase-correct PWM, which is what the 31.37 kHz carrier calculation assumes.

---

## Build and test

Power-up checks:
1. Measure the 12 V supply with a multimeter before connecting it. It must be 12 V or less. Cheap unregulated adapters can give much more when lightly loaded.
2. Check electrolytic polarity, and that LM386 pin 6 is +12 V and pins 2 and 4 are GND.
3. Connect the load between the phases (see [Output stage and the load](#output-stage-and-the-load)) before powering the amplifiers.

Step by step:
1. Upload the sketch. Scope pin 9 directly: a 31 kHz pulse train with changing width.
2. Scope the RC filter node: a 20 Hz sine of about +-2.4 V on 2.5 V DC.
3. Scope after the 10 uF: the same sine centred on 0 V.
4. Power the LM386 stages and scope each output (AC coupling).
5. Compare the three phases. Trigger on phase 1 and check that the other two are shifted by one third of a period.

---

## Simulation (Proteus)

Files are in `simulation/` (adjust the path).

- Load the compiled `.hex` into the Nano and run the simulation.
- Use scope probes to measure. **Do not use ammeters or voltmeters in the AC path.** A zero-ohm ammeter next to capacitors caused the SPICE error "timestep too small" until it was removed.
- If a node has no DC path to ground (for example a floating star point), add a 10 Mohm resistor from it to GND.
- If the simulation is slow, test at 200-250 Hz with a shorter time base, then return to 20-25 Hz.
- Proteus models can be more generous than real silicon. Treat the simulated amplitude as an upper bound.

---

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| No sine at the RC filter | Wrong hex, wrong pins, or the Nano is not running the sketch |
| Flat tops on the output | Divider ratio too high (raise the 10k series resistor to 15k or 22k) |
| High-frequency buzz, chip hot | Missing Zobel network or supply decoupling, or long wires |
| Chips warm, large DC current in the load | Load or star point connected to GND |
| Unequal amplitudes | Different part values or loads between phases |
| One phase shows nothing | Check the filter, coupling capacitor polarity and the pin 3 connection |
| Proteus "timestep too small" | Remove ammeters, add a DC path to GND |

---

## Limits and safety

- This is a **linear, single-ended, low-power** source. It is not a motor drive and not an inverter.
- The LM386 is rated for about 325 mW (4-12 V). Keep the supply at 12 V or below, and do not short an output.
- For real power or a real motor, use a MOSFET half-bridge (for example IR2104 gate drivers) with an LC output filter.

---

## Repository layout

```
.
├── hardware/       PCB (schematic, layout, Gerbers)
├── simulation/     Proteus project
├── docs/           Photos and scope captures
└── README.md
```

---


## Credits

- Firmware: [cmasenas/3-Phase-Sine-Arduino](https://github.com/cmasenas/3-Phase-Sine-Arduino) by C. Masenas, modified from DDS code by KHM 2009 / Martin Nawrath.
- Hardware design, PCB and simulation by Ahmed, Faculty of Engineering, Alexandria University.

## License

MIT License
