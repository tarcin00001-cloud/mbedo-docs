# 41 - ESP32 Water Level Analog Level Print

Read the analog output of a water level sensor module and print the level as a percentage and descriptive label — building the foundation for a tank monitoring system.

## Goal
Learn how to read a water level sensor with the ESP32 ADC, map the raw reading to a meaningful 0–100% scale, and log descriptive tank status messages.

## What You Will Build
A water level sensor module with its analog output connected to GPIO 34. The code reads the ADC continuously, maps it to a fill percentage, and classifies the tank as Empty, Low, Half, High, or Full.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Water Level Sensor Module | `water_sensor` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Level Sensor | VCC | 3V3 | Red | Module supply voltage |
| Water Level Sensor | GND | GND | Black | Module ground |
| Water Level Sensor | SIG (AO) | GPIO34 | Yellow | Analog level output |

> **Wiring tip:** Only power the water sensor when taking a reading — continuous current through the sensing traces causes electrolytic corrosion over time. For long-duration deployments, connect VCC through a GPIO output pin (e.g. GPIO 26) and set it HIGH only during the ADC read, then LOW again. GPIO 34 on the ESP32 is input-only and ideal for this ADC task. The sensor reads higher ADC values as more traces are submerged.

## Code
```cpp
// Water Level Analog Level Print
const int WATER_PIN = 34;

// Calibration — measure actual ADC values in empty/full states
const int EMPTY_VAL = 0;     // ADC when no traces submerged
const int FULL_VAL  = 3200;  // ADC when all traces submerged

String classifyLevel(int pct) {
  if      (pct < 10) return "EMPTY — refill now!";
  else if (pct < 30) return "LOW   — fill soon";
  else if (pct < 60) return "HALF  — adequate";
  else if (pct < 85) return "HIGH  — good level";
  else               return "FULL  — maximum";
}

void setup() {
  Serial.begin(115200);
  Serial.println("Water Level Sensor ready.");
}

void loop() {
  int raw  = analogRead(WATER_PIN);

  // Map raw to percentage
  int level = map(raw, EMPTY_VAL, FULL_VAL, 0, 100);
  level = constrain(level, 0, 100);

  Serial.print("ADC: "); Serial.print(raw);
  Serial.print("  |  Level: "); Serial.print(level);
  Serial.print("%  |  Status: ");
  Serial.println(classifyLevel(level));

  delay(800);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Water Level Sensor** onto the canvas.
2. Connect Sensor **SIG** to **GPIO34**.
3. Paste the code and click **Run**.
4. Drag the water level slider widget from empty to full — observe the percentage and label update.

## Expected Output
Serial Monitor:
```
Water Level Sensor ready.
ADC: 50    |  Level: 1%   |  Status: EMPTY — refill now!
ADC: 640   |  Level: 20%  |  Status: LOW   — fill soon
ADC: 1600  |  Level: 50%  |  Status: HALF  — adequate
ADC: 2560  |  Level: 80%  |  Status: HIGH  — good level
ADC: 3200  |  Level: 100% |  Status: FULL  — maximum
```

## Expected Canvas Behavior
* Moving the water level slider from the bottom to the top smoothly increases the percentage readout.
* The status label changes as the slider crosses each classification threshold.
* Logging occurs every 800 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int EMPTY_VAL = 0` | ADC reading when no traces are in water — the 0% calibration point. |
| `const int FULL_VAL = 3200` | ADC reading when all sensing traces are fully submerged — the 100% calibration point. |
| `map(raw, EMPTY_VAL, FULL_VAL, 0, 100)` | Scales the ADC count to a 0–100% range based on calibration bounds. |
| `constrain(level, 0, 100)` | Prevents values outside 0–100 if the sensor reading drifts beyond calibration bounds. |
| `classifyLevel(level)` | Returns a status string based on five percentage bands. |

## Hardware & Safety Concept: Resistive Water Level Sensing
A water level sensor contains a series of exposed conductive traces at different heights. The more traces that are submerged in water, the lower the total resistance across the sensing strip (parallel resistance). The module converts this resistance change into an analog voltage output using an internal voltage divider. As the water rises and submerges more traces, the output voltage increases proportionally. The relationship is approximately linear with the number of submerged traces, making `map()` a reasonable conversion. In industrial applications, more accurate methods include **ultrasonic distance measurement** (Project 75–77), **pressure transducers** (measures water column pressure), and **float switches** (simple mechanical level detection for alarm purposes).

## Try This! (Challenges)
1. **Pump control**: Add a relay on GPIO 5. Turn the pump ON below 20% and OFF above 80%.
2. **LED bar graph**: Light LEDs on GPIOs 12–15 to represent 0%, 33%, 66%, and 100% fill levels.
3. **Low-water buzzer alarm**: Sound a buzzer on GPIO 25 with 3 short beeps when the level drops below 10%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads 0 or very low | SIG pin not connected to GPIO 34 | Re-seat the SIG wire; verify module is powered |
| Level does not reach 100% | FULL_VAL calibration too high for your sensor | Submerge the sensor fully and read the actual ADC value — use that as FULL_VAL |
| Level drifts even in still water | Long SIG wire or capacitive noise | Shorten the GPIO 34 wire; add a 100 nF capacitor from GPIO 34 to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [40 - ESP32 Soil Moisture Analog Level Print](40-esp32-soil-moisture-analog-level-print.md)
- [39 - ESP32 Rain Sensor Digital Alert](39-esp32-rain-sensor-digital-alert.md)
- [31 - ESP32 Potentiometer ADC Read](31-esp32-potentiometer-adc-read.md)
