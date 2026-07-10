# 43 - ESP32 MQ-2 Gas Sensor Level Print

Read the analog output of an MQ-2 gas sensor module and print the raw voltage and a qualitative gas concentration level to the Serial Monitor.

## Goal
Learn how the MQ-2 gas sensor detects LPG, smoke, hydrogen, and methane, how to read its analog output, and how to apply threshold levels to classify air quality.

## What You Will Build
An MQ-2 module with its analog output connected to GPIO 34. The code reads the sensor after a warm-up period, converts the ADC to a voltage, and classifies the air as Clean, Low Gas, Moderate Gas, or High Gas Alert.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `gas_sensor` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Module | VCC | 5V (Vin) | Red | Sensor heater requires 5 V |
| MQ-2 Module | GND | GND | Black | Common ground |
| MQ-2 Module | AO (analog out) | GPIO34 | Yellow | Analog gas concentration voltage |

> **Wiring tip:** The MQ-2 heater coil requires **5 V at ~150 mA** — connect VCC to the ESP32's Vin or an external 5 V rail, NOT to 3V3. The AO output is scaled to 0–5 V by the onboard voltage divider; however, most modules include a voltage divider that limits AO to ~0–3.3 V for microcontroller compatibility — verify your specific module. Allow **30–60 seconds warm-up** before readings are reliable. High ADC values indicate higher gas concentration.

## Code
```cpp
// MQ-2 Gas Sensor Level Print
const int MQ2_PIN = 34;

// Approximate ADC thresholds (calibrate in clean air first)
const int CLEAN_MAX    = 400;   // Below this = clean air
const int LOW_GAS_MAX  = 1200;  // Low gas detection
const int MOD_GAS_MAX  = 2400;  // Moderate concentration

String classifyGas(int raw) {
  if      (raw < CLEAN_MAX)   return "Clean air";
  else if (raw < LOW_GAS_MAX) return "Low gas detected";
  else if (raw < MOD_GAS_MAX) return "Moderate — ventilate area";
  else                        return "!! HIGH GAS ALERT !!";
}

void setup() {
  Serial.begin(115200);
  Serial.println("MQ-2 Gas Sensor warming up (30 s)...");
  delay(30000);   // Warm-up period
  Serial.println("Ready — monitoring air quality.");
}

void loop() {
  int raw     = analogRead(MQ2_PIN);
  float volts = raw * (3.3f / 4095.0f);

  Serial.print("ADC: "); Serial.print(raw);
  Serial.print("  |  "); Serial.print(volts, 2); Serial.print(" V");
  Serial.print("  |  "); Serial.println(classifyGas(raw));

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **MQ-2 Gas Sensor** onto the canvas.
2. Connect Sensor **AO** to **GPIO34**.
3. Paste the code and click **Run**.
4. Drag the gas sensor slider widget to simulate different gas concentrations.

## Expected Output
Serial Monitor:
```
MQ-2 Gas Sensor warming up (30 s)...
Ready — monitoring air quality.
ADC: 280   |  0.23 V  |  Clean air
ADC: 900   |  0.73 V  |  Low gas detected
ADC: 1800  |  1.45 V  |  Moderate — ventilate area
ADC: 3100  |  2.50 V  |  !! HIGH GAS ALERT !!
```

## Expected Canvas Behavior
* At the low end of the slider, Serial Monitor shows "Clean air."
* Moving the slider up increases the ADC reading and triggers higher alert classifications.
* The warm-up delay is simulated quickly in MbedO.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `delay(30000)` | 30-second warm-up period — MQ sensors need time for the heater to stabilise the sensitive metal oxide layer. |
| `int raw = analogRead(MQ2_PIN)` | Reads the gas concentration as a 12-bit ADC count. |
| `float volts = raw * (3.3f / 4095.0f)` | Converts the raw count to a voltage. |
| `classifyGas(raw)` | Compares the ADC count to threshold bands and returns a descriptive status. |

## Hardware & Safety Concept: Metal Oxide Gas Sensors (MEMS)
The MQ-2 sensor contains a small heating element and a tin dioxide (SnO₂) metal oxide semiconductor layer. In clean air at operating temperature (~200–300 °C, heated internally), SnO₂ has high resistance. When combustible gases (LPG, CH₄, H₂) or smoke contact the hot surface, they reduce the surface oxygen layer, dramatically lowering the semiconductor resistance. This resistance decrease is converted to a rising output voltage by the module's voltage divider. MQ sensors are cross-sensitive — MQ-2 responds to LPG, propane, methane, hydrogen, smoke, and alcohol. They cannot distinguish individual gases. For specific gas identification, electrochemical sensors (NDIR CO₂, PEM H₂) or MEMS spectrometers are required. MQ sensors are widely used in home gas alarms, industrial safety monitors, and kitchen ventilation controllers.

## Try This! (Challenges)
1. **Relay cutoff**: Wire a relay on GPIO 5 to cut power to a gas valve when the HIGH GAS ALERT level is reached.
2. **Buzzer alarm**: Sound a buzzer on GPIO 15 with increasing urgency — slow beeps at "Moderate", fast beeps at "High".
3. **Baseline calibration**: In clean air, record the average ADC value over 60 seconds; use that + 200 as your CLEAN_MAX threshold.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads HIGH GAS ALERT | Sensor not warmed up or VCC too low | Wait full warm-up time; verify 5 V VCC connection |
| Reads very low even near gas | AO pin wired to wrong GPIO | Confirm AO connects to GPIO 34 (not DO) |
| No change when gas is present | Module heater not powered | Check 5 V supply current can supply 150 mA for the heater |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [39 - ESP32 Rain Sensor Digital Alert](39-esp32-rain-sensor-digital-alert.md)
- [44 - ESP32 Flame Sensor Digital Fire Alert](44-esp32-flame-sensor-digital-fire-alert.md)
- [50 - ESP32 Sound Sensor Threshold Alarm](50-esp32-sound-sensor-threshold-alarm.md)
