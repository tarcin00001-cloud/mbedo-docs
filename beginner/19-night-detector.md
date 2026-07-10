# 19 - Night Detector

Create an automated night light that turns an LED on when it gets dark.

## Goal
Learn how to use an LDR's analog reading to trigger a digital output (LED) dynamically based on a custom threshold limit.

## What You Will Build
When the ambient light level drops below a set threshold (simulating nightfall), the LED turns on automatically. When light levels rise, the LED turns off.

**Why A0 and D13?** Pin A0 tracks the analog light levels continuously. Pin D13 drives the visual indicator LED (simulating a street lamp).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| LDR Sensor | VCC | 5V | Power supply (5V) |
| LDR Sensor | AO | A0 | Analog signal connection |
| LDR Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int LDR_PIN = A0;
const int LED_PIN = 13;

// Threshold below which we consider it "dark" (0 to 1023)
const int DARK_THRESHOLD = 400;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Night Detector Ready");
}

void loop() {
  int lightLevel = analogRead(LDR_PIN);
  
  Serial.print("Light Level: ");
  Serial.println(lightLevel);
  
  // If the light level is below our threshold, turn ON the LED
  if (lightLevel < DARK_THRESHOLD) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Night detected - LED ON");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **LDR Light Sensor**, and **LED** onto the canvas.
2. Connect LDR **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the LDR Sensor on the canvas, move the slider to low light (below 400), and watch the LED turn on.

## Expected Output

Terminal:
```
Night Detector Ready
Light Level: 800
Light Level: 350
Night detected - LED ON
...
```

### Expected Canvas Behavior

| Ambient Light State | Pin A0 Reading | Condition check | LED State (D13) |
| --- | --- | --- | --- |
| Bright (Day) | > 400 | False | LOW - OFF |
| Dark (Night) | < 400 | True | HIGH - ON |

The LED activates immediately when the slider falls below the threshold.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `const int DARK_THRESHOLD = 400;` | Defines our switching point. Lower values represent darkness, and higher values represent daylight. |
| `if (lightLevel < DARK_THRESHOLD)` | Evaluates if the current light level is below the threshold. If true, the LED is driven HIGH. |

## Hardware & Safety Concept: Threshold Hysteresis
In real-world automated systems, if the light level hovers exactly at the threshold value (e.g. 400), the LED might flicker on and off rapidly due to minor voltage noise. Real-world systems use **hysteresis** (e.g., turn ON at 380, but do not turn OFF until it reaches 420) to prevent this rapid switching and protect the hardware.

## Try This! (Challenges)
1. **Calibrate Threshold**: Calibrate the system by finding the exact slider value where you want the light to activate, and update `DARK_THRESHOLD` in the code.
2. **Reverse logic (Solar Indicator)**: Change the logic so the LED turns ON when it is *bright* (above 800) and turns OFF when it is dark, simulating a solar battery charging indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on even in bright light | Threshold is set too high | Lower the `DARK_THRESHOLD` constant (e.g., from 400 to 200). |
| LED never turns on | Threshold is set too low or wiring error | Raise the `DARK_THRESHOLD` (e.g. to 600) or check the LED wiring. |

## Mode Notes
Conditional testing of an `analogRead` result against a constant value is supported by MbedO interpreted mode and is safe for this beginner project.

## Related Projects
- [18 - Light Meter](18-light-meter.md)
- [30 - Motion Light](../intermediate/30-motion-light.md)
