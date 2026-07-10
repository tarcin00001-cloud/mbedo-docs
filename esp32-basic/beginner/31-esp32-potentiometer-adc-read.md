# 31 - ESP32 Potentiometer ADC Read

Read the analog voltage output of a potentiometer using the ESP32's 12-bit ADC and print the raw value and the calculated voltage to the Serial Monitor.

## Goal
Learn how to use `analogRead()` on the ESP32 to measure a continuously varying voltage, understand the 12-bit resolution of the ESP32 ADC, and convert raw counts to a voltage value.

## What You Will Build
A potentiometer with its wiper connected to GPIO 34 (ADC1_CH6). Turning the knob changes the voltage from 0 V to 3.3 V. The code reads the 12-bit ADC value (0–4095) and prints both the raw count and the mapped voltage every 300 ms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left leg (VCC end) | 3V3 | Red | Reference voltage supply |
| Potentiometer | Right leg (GND end) | GND | Black | Ground reference |
| Potentiometer | Wiper (middle leg) | GPIO34 | Yellow | Analog signal output |

> **Wiring tip:** GPIO 34 on the ESP32 is an input-only pin with no internal pull resistors — ideal for ADC use. Never connect the wiper to a voltage above 3.3 V; the ESP32's ADC inputs are not 5 V tolerant. The left and right legs can be swapped to reverse the direction of rotation if preferred.

## Code
```cpp
// Potentiometer ADC Read — 12-bit ESP32 ADC
const int POT_PIN = 34;   // ADC1_CH6 — input-only GPIO

void setup() {
  Serial.begin(115200);
  Serial.println("Potentiometer ADC Reader ready.");
  // analogRead resolution defaults to 12-bit (0–4095) on ESP32
}

void loop() {
  int rawValue = analogRead(POT_PIN);    // 0–4095

  // Convert raw ADC count to voltage: V = raw * (3.3 / 4095)
  float voltage = rawValue * (3.3f / 4095.0f);

  Serial.print("ADC Raw: ");
  Serial.print(rawValue);
  Serial.print("  |  Voltage: ");
  Serial.print(voltage, 2);   // 2 decimal places
  Serial.println(" V");

  delay(300);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Connect Potentiometer **wiper** to **GPIO34**.
3. Paste the code and click **Run**.
4. Drag the potentiometer slider widget and observe the ADC values changing in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Potentiometer ADC Reader ready.
ADC Raw: 0     |  Voltage: 0.00 V
ADC Raw: 1024  |  Voltage: 0.82 V
ADC Raw: 2048  |  Voltage: 1.65 V
ADC Raw: 4095  |  Voltage: 3.30 V
```

## Expected Canvas Behavior
* Dragging the potentiometer slider to the left end prints ADC Raw: 0 and Voltage: 0.00 V.
* Dragging to the right end prints ADC Raw: 4095 and Voltage: 3.30 V.
* Intermediate positions print proportional values.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `int rawValue = analogRead(POT_PIN)` | Samples the ADC and returns an integer from 0 (0 V) to 4095 (3.3 V). |
| `float voltage = rawValue * (3.3f / 4095.0f)` | Scales the 12-bit count to a voltage in volts using the 3.3 V reference. |
| `Serial.print(voltage, 2)` | Prints the float with 2 decimal places for a clean display. |
| `delay(300)` | Samples at ~3 Hz — slow enough to read on the Serial Monitor without flooding it. |

## Hardware & Safety Concept: ESP32 12-bit ADC Resolution
The ESP32 contains a 12-bit successive-approximation register (SAR) ADC that divides the 0–3.3 V input range into 4096 discrete steps (2¹² = 4096). Each step represents 3.3 V ÷ 4095 ≈ **0.806 mV** of resolution. This is 16× finer than the Arduino Uno's 10-bit ADC (which has 1024 steps ≈ 4.88 mV per step). Note that the ESP32 ADC has a known non-linearity near 0 V and 3.3 V (the readings compress at the extremes). For precision voltage measurement, the ESP32 ADC is best used in the 0.1 V–3.1 V range. For production-grade accuracy, an external ADC chip (e.g. ADS1115) is recommended.

## Try This! (Challenges)
1. **Percentage display**: Map the raw ADC value to 0–100% and print it alongside the voltage.
2. **Min/Max tracker**: Track the minimum and maximum readings seen since startup and print them.
3. **Voltage alarm**: Print "OVER VOLTAGE WARNING" if the wiper voltage exceeds 2.8 V.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads 0 or 4095 | Wiper not connected to GPIO 34 | Verify the wiper (middle) leg is wired to GPIO 34 |
| Reading jumps erratically | Long wiper wire picking up noise | Shorten the wire to GPIO 34; add a 100 nF capacitor from GPIO 34 to GND |
| Voltage calculation seems wrong | Wrong divisor in formula | Use 4095.0 (not 4096.0) — the maximum raw value is 4095 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - ESP32 Potentiometer LED Dimmer](32-esp32-potentiometer-led-dimmer.md)
- [33 - ESP32 LDR Light Intensity Meter](33-esp32-ldr-light-intensity-meter.md)
- [47 - ESP32 Analog Joystick X-Axis Logging](47-esp32-analog-joystick-x-axis-logging.md)
