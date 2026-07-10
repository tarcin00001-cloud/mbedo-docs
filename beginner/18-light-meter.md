# 18 - Light Meter

Measure ambient light levels using a Light Dependent Resistor (LDR) and display them.

## Goal
Learn how to read an analog light sensor, understand how photoresistors behave under light, and print raw sensor values to the Terminal.

## What You Will Build
As you change the light level slider on the LDR sensor, the Terminal prints the updated raw reading (0 to 1023) every 200 ms.

**Why A0?** The LDR sensor output is an analog voltage that must be read by the Arduino's ADC. Pin A0 is configured to perform this conversion.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (required for discrete wiring) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| LDR Sensor | VCC | 5V | Power supply (5V) |
| LDR Sensor | AO (Analog Out) | A0 | Analog signal connection |
| LDR Sensor | GND | GND | Ground reference |

## Code
```cpp
const int LDR_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("LDR Light Meter Ready");
}

void loop() {
  // Read raw light sensor value (0 to 1023)
  int lightLevel = analogRead(LDR_PIN);
  
  Serial.print("Light Level (Raw): ");
  Serial.println(lightLevel);
  
  delay(200); // Read 5 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **LDR Light Sensor** onto the canvas.
2. Connect LDR **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the LDR sensor on the canvas, adjust the ambient light slider, and watch the values change in the Terminal.

## Expected Output

Terminal:
```
LDR Light Meter Ready
Light Level (Raw): 120
Light Level (Raw): 540
Light Level (Raw): 980
...
```

### Expected Canvas Behavior

| Ambient Light Slider | Sensor Resistance | Signal Voltage | Pin A0 Reading |
| --- | --- | --- | --- |
| Dark | Very High | Near 0V | Low (e.g. < 200) |
| Indoor Light | Moderate | Near 2.5V | Medium (e.g. 500) |
| Direct Sunlight | Very Low | Near 5.0V | High (e.g. > 900) |

The raw output matches the slider value directly in real-time.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `analogRead(LDR_PIN)` | Measures the analog voltage output of the sensor module and returns an integer from 0 (darkest) to 1023 (brightest). |

## Hardware & Safety Concept: Photoresistors (LDRs)
A **Light Dependent Resistor (LDR)** changes its electrical resistance based on the amount of light falling on its surface.
- In darkness, the resistance is extremely high (often exceeding 1 Megaohm).
- In bright light, the resistance drops significantly (down to a few hundred ohms).
By combining the LDR with a fixed resistor in a voltage divider circuit on the sensor breakout board, this changing resistance is converted to a variable voltage that the Arduino ADC can measure.

## Try This! (Challenges)
1. **Light Percentage**: Scale the output using `map(lightLevel, 0, 1023, 0, 100)` to display a percentage scale from 0% (dark) to 100% (bright).
2. **Low Light Alert**: Add an `if` statement to print a warning message "Night Mode Active" to the Terminal when the light level falls below 300.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings are stuck at 0 or 1023 | Wiring issue | Make sure LDR VCC is connected to 5V, GND to GND, and AO to A0. |
| Values do not change when slider is adjusted | Wrong pin configured in code | Ensure the `LDR_PIN` constant matches the A0 pin you wired the sensor to. |

## Mode Notes
These patterns (analog input and serial logging) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
- [19 - Night Detector](19-night-detector.md)
