# 110 - Pico Irrigation Station

Build an automated agricultural node that switches a water valve relay based on analog soil moisture levels.

## Goal
Learn how to read analog soil moisture sensors, map wet/dry thresholds, and trigger irrigation solenoid relays.

## What You Will Build
An automatic soil watering system:
- **Soil Moisture Sensor (GP26)**: Measures moisture levels in soil.
- **Relay Module (GP10)**: Actuates a 5V solenoid water valve to irrigate soil when moisture levels drop below the dry threshold.
- **Status LED (GP15)**: Glows when watering is active.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes (resistive/capacitive probe) |
| Relay Module | `relay` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Soil Sensor | VCC | 3.3V | Power supply |
| Soil Sensor | AO (Analog) | GP26 | Moisture level signal |
| Soil Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Valve control line |
| Relay Module | GND | GND | Ground return |
| Red LED | Anode | GP15 | Activity indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int SOIL_PIN  = 26;
const int RELAY_PIN = 10;
const int LED_PIN   = 15;

// Moisture thresholds (raw ADC readings)
// Dry soil reads high voltage (e.g. >3000). Wet soil reads low voltage (<1500).
const int DRY_LIMIT = 2800;

void setup() {
  pinMode(SOIL_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with valve closed
  digitalWrite(LED_PIN, LOW);
}

void loop() {
  int moisture = analogRead(SOIL_PIN);

  // If soil moisture falls below safety limit (meaning sensor value is high)
  if (moisture > DRY_LIMIT) {
    digitalWrite(RELAY_PIN, HIGH); // Open water valve
    digitalWrite(LED_PIN, HIGH);   // Turn activity LED ON
    delay(4000);                   // Water for 4 seconds
    digitalWrite(RELAY_PIN, LOW);  // Close water valve
    digitalWrite(LED_PIN, LOW);
    delay(10000);                  // Wait 10 seconds to let water soak in
  } else {
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(LED_PIN, LOW);
  }

  delay(1000); // Check once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Soil Moisture Sensor** (represented by potentiometer), **Relay Module**, and **Red LED** onto the canvas.
2. Connect Soil Sensor to **GP26**, Relay to **GP10**, and LED to **GP15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the moisture slider to simulate dry soil (high value) and watch the relay open the valve for 4 seconds.

## Expected Output

Terminal:
```
Simulation active. Soil moisture monitoring loop active.
```

## Expected Canvas Behavior
| Moisture level slider | GP10 (Relay state) | GP15 (LED state) | Watering Status |
| --- | --- | --- | --- |
| Wet (< 1500) | LOW | LOW | Safe (No action) |
| Dry (> 2800) | HIGH (Closed) | HIGH (ON) | **Active (Watering for 4s)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `delay(10000)` | Soak delay preventing the system from watering again immediately, allowing water to filter down to the roots and update the sensor reading. |

## Hardware & Safety Concept: Sensor Corrosion Prevention
Resistive soil moisture probes pass electrical current through two exposed metal traces in the dirt. Keeping them powered constantly causes rapid **electrolysis corrosion** that degrades the metal traces in weeks. To extend sensor life, power the sensor from a digital GPIO pin (e.g. GP16) rather than 3.3V VCC. Turn the GPIO pin `HIGH` only during measurement, read the analog value, and write the GPIO pin `LOW` immediately after.

## Try This! (Challenges)
1. **Critical Low Alarm**: Sound a buzzer on GP14 if the moisture level remains critical after three consecutive watering cycles.
2. **Moisture HUD**: Connect a 16x2 LCD display on GP4/GP5 and print moisture percentages.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay oscillates ON and OFF constantly | Soak delay missing | Ensure the code includes a post-watering delay (`delay(10000)`) to give water time to absorb and reach the sensor probes. |

## Mode Notes
This multi-device analog control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [40 - Pico Soil Moisture](../../beginner/40-pico-soil-moisture.md)
- [102 - Pico Water Station](102-pico-water-station.md)
