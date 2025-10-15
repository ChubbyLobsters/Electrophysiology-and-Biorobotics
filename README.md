# Electrophysiology-and-Biorobotics
Computational Electrophysiology in Plants: A Low-Cost Arduino Approach

# INA126PA + Arduino Uno — Plant Bio-Signal Amplifier

> DIY project: measure small electrical signals from plants (µV → mV) using an **INA126PA** instrumentation amplifier and an **Arduino Uno**.  
> Ready-to-paste `README.md` for your GitHub repo.

---

## Links
- INA126PA (Jaycar): https://www.jaycar.co.nz/ic-ina126pa-micropower-instrumentation-amplifier/p/ZL3976

---

## Project Summary

This project shows how to wire an **INA126PA** instrumentation amplifier to an **Arduino Uno** to measure tiny differential voltages from plant tissues. The INA126 provides high input impedance, adjustable gain with a single RG resistor, and excellent CMRR — ideal for amplifying plant bio-signals. Output is level-shifted around a VREF (mid-rail) so the Arduino ADC (0–5 V) can read signals that swing above and below baseline.

---

## Shopping list

- **IC: INA126PA Micropower Instrumentation Amplifier** — Jaycar: <https://www.jaycar.co.nz/ic-ina126pa-micropower-instrumentation-amplifier/p/ZL3976>  
- Resistors
  - `1 × 1 kΩ` (RG — start here; adjust for gain)
  - `2 × 100 kΩ` (voltage divider for VREF ≈ 2.5 V)
  - Optional: `1 × 10 kΩ` (for optional output RC filter)
- Capacitors
  - `1 × 10 µF` electrolytic (VREF decoupling)
  - `1 × 0.1 µF` ceramic (supply decoupling, place close to IC)
  - Optional: extra `0.1 µF` for output RC
- Electrodes & wiring
  - Copper tape / aluminium foil / ECG pads
  - Short shielded/twisted jumper wires
- Breadboard & jumper wires
- Arduino Uno (you already have)
- Optional: rail-to-rail op amp (e.g., TLV2372) to buffer VREF (recommended)
- Optional: battery pack (for quieter operation)
- Tools: multimeter, solderless breadboard, small screwdriver

---

## Quick facts & formula

- **Gain (INA126):**

\[
G = 5 + \frac{80\,000}{R_G}
\]

where `R_G` is in ohms. Example: `R_G = 1 kΩ` → `G = 85`.

- Arduino Uno ADC: **10-bit** (0..1023), default Vref = `5.0 V`.

---

## Breadboard wiring (ASCII schematic — simplified)

```
Arduino 5V ----+------------------------------+-------------------+
               |                              |                   |
             100k R1                      INA126 pin7 (V+)       |
               |                              |                   |
           VREF node ~2.5V ----+---- 10uF cap  | INA126 pin6(VO) --- (to Arduino A0)
               |               |              |                   |
             100k R2           |           INA126 pin5(REF)       |
               |               |              |                   |
Arduino GND ---+---------------+--------------+---- INA126 pin4 (V- / GND)
                                              |
                                   INA126 pin3 (+IN) <-- Electrode A (plant)
                                   INA126 pin2 (−IN) <-- Electrode B (plant)
                                   INA126 pin1(RG) ---[ 1kΩ ]--- pin8(RG)

Notes:
- Put 0.1µF cap between V+ (pin7) and V− (pin4) as close to the IC as possible.
- VREF = midpoint of two 100k resistors (5V ↔ 100k ↔ VREF ↔ 100k ↔ GND).
- Add 10µF from VREF → GND and 0.1µF near REF pin for decoupling.
- Optional output filter: series 10kΩ + 0.1µF to GND (≈ 159 Hz cutoff).
```

---

## Wiring checklist (step-by-step)

1. Insert INA126 (DIP) across breadboard center gap (pins 1–4 one side, 5–8 other).  
2. Connect **pin 7 (V+)** → **Arduino +5V**.  
3. Connect **pin 4 (V−)** → **Arduino GND**.  
4. Create **VREF**:
   - Series 100 kΩ between 5 V and GND; midpoint ≈ **VREF = 2.5 V**.
   - Add `10 µF` (electrolytic) VREF → GND and `0.1 µF` ceramic near REF pin.
   - Connect **pin 5 (REF)** → **VREF** (or buffered VREF output).
5. Place **RG = 1 kΩ** between **pin1** and **pin8** (gain ≈ 85).  
6. Connect **pin3 (+IN)** → **Electrode A**; **pin2 (−IN)** → **Electrode B**.  
7. Connect **pin6 (VO)** → **Arduino A0**. (Optional: series 10 kΩ + 0.1 µF to GND for RC filter.)  
8. Add `0.1 µF` decoupling between pin7 and pin4 near the IC.  
9. Power Arduino (battery recommended). Measure VREF with a multimeter and update code accordingly.

---

## How it works (plain explanation)

1. **Electrodes** on the plant pick up a tiny differential voltage (microvolts to millivolts).  
2. **INA126** senses `(VIN+ − VIN−)` with very high input impedance and amplifies it by `G` set with `R_G`.  
3. The INA output is offset by `VREF` (≈ 2.5 V) so small positive/negative swings map inside Arduino ADC range (0–5 V).  
4. Arduino reads `Vout`. Software computes:

```
Vin_diff = (Vout - VREF) / G
Vin_uv   = Vin_diff * 1e6   // microvolts
```

---

## Arduino sketch (copy & paste)

```cpp
// INA126 + Arduino Uno example
const int sensorPin = A0;
const float VCC = 5.0;           // Arduino supply
const float VREF_measured = 2.50; // measure the actual midpoint with multimeter
const float RG = 1000.0;         // ohms (1k -> gain ≈ 85)
const float GAIN = 5.0 + 80000.0 / RG;
const int ADC_MAX = 1023;

void setup() {
  Serial.begin(115200);
  delay(200);
  Serial.println("INA126 Plant Read (uV)");
  Serial.print("Gain = "); Serial.println(GAIN);
  Serial.print("VREF = "); Serial.println(VREF_measured, 4);
}

void loop() {
  int raw = analogRead(sensorPin);              // 0..1023
  float vout = raw * (VCC / (float)ADC_MAX);    // volts at INA output
  float vin_diff = (vout - VREF_measured) / GAIN; // differential input volts
  float vin_uv = vin_diff * 1e6;                // microvolts

  Serial.print(millis()); Serial.print(" ms, ");
  Serial.print("raw: "); Serial.print(raw);
  Serial.print("  Vout: "); Serial.print(vout, 6);
  Serial.print("  Vin_diff (uV): "); Serial.println(vin_uv, 2);

  delay(100); // 10 Hz sampling (adjust as needed)
}
```

---

## Practical tips & troubleshooting

- Start with `R_G = 1 kΩ`. If `Vout` is saturated at 0 V or 5 V, reduce gain (increase `R_G`).  
- Keep electrode wires short & twisted; use shielded cable if possible.  
- Battery power reduces mains hum compared to USB.  
- Buffer `VREF` with a rail-to-rail op amp (voltage follower) for lower impedance & improved CMRR.  
- Wet electrode contacts (saline or aloe) to improve conductivity; ECG pads are ideal.  
- Use software averaging (moving average) to smooth slow plant signals.  
- Use single common ground to avoid ground loops.  
- If no signal: try different electrode placement (same leaf, leaf-to-stem, soil reference), wet contacts, and verify wiring.

---

## Safety & ethics

- Measured voltages are extremely small — **no shock hazard**. Do **not** connect mains-powered or uninsulated mains parts to electrodes.  
- Use inert/disposable electrodes to avoid plant damage. Follow your university's ethics and safety guidelines for living subject experiments.

---

## Next steps / enhancements

- Buffer VREF with a rail-to-rail op amp for better performance.  
- Add instrumentation amplifier output buffering / anti-alias filtering.  
- Log data to SD card or stream to PC for plotting (Processing, Python + matplotlib).  
- Create a small PCB breakout for INA126 + RG + VREF buffer for reproducible results.

---

## License & attribution

Feel free to copy and adapt this README for your project. If you publish or present results, consider referencing sources for INA126 datasheet and instrumentation amplifier application notes.
