# 24 - Pico Limit Switch

Read limit switch actuation states to log system boundary alerts.

## Goal
Learn how to interface mechanical limit switches with digital inputs on the Raspberry Pi Pico to detect machine movement boundaries.

## What You Will Build
A boundary limit safety indicator:
- **Limit Switch (GP16)**: Configured with an internal pull-up resistor (active-LOW).
- **Warning LED (GP15)**: Turns ON (Red alert) immediately when the limit switch arm is depressed, indicating a collision or mechanical boundary reach.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Limit Switch (SPDT) | `button` | Yes (or switch module) | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Limit Switch | COM (Common) | GP16 | Digital sense pin |
| Limit Switch | NO (Normally Open)| GND | Ground connection |
| Red LED | Anode | GP15 | Safety warning indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int LIMIT_PIN = 16;
const int LED_PIN   = 15;

void setup() {
  pinMode(LIMIT_PIN, INPUT_PULLUP); // Enable internal pull-up on COM pin
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int limitState = digitalRead(LIMIT_PIN);

  // Active-LOW check: COM connected to NO (GND) when arm is hit
  if (limitState == LOW) {
    digitalWrite(LED_PIN, HIGH); // Boundary reached! Alert ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Normal state, Alert OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Limit Switch** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Switch: **COM** pin to **GP16**, **NO** pin to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Depress the limit switch arm to trigger the warning light.

## Expected Output

Terminal:
```
Simulation active. GP16 limit safety monitor online.
```

## Expected Canvas Behavior
| Limit Switch State | GP16 Voltage | LED state (GP15) | Safety Status |
| --- | --- | --- | --- |
| Not depressed (Open)| 3.3V (HIGH) | LOW (OFF) | Normal |
| Depressed (Closed) | 0V (LOW) | HIGH (ON) | **ALERT (Boundary hit)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pinMode(LIMIT_PIN, INPUT_PULLUP)` | Configures GP16 as an input, holding it HIGH via the internal resistor until the switch arm pulls it to GND. |

## Hardware & Safety Concept: Limit Switch Wiring (NC vs. NO)
For safety critical emergency stops, limit switches are wired to their **Normally Closed (NC)** contacts. This means the pin reads `LOW` during normal operation and goes `HIGH` (floating) if a wire breaks. This "fail-safe" design guarantees that a damaged or cut wire triggers an immediate machine shutdown, rather than leaving the machine unable to detect a boundary crash.

## Try This! (Challenges)
1. **Safety Interlock**: Connect a buzzer to GP14 and sound a repeating alarm if the limit switch remains depressed.
2. **Reverse Fail-Safe Logic**: Wire the switch to NC and rewrite the code to turn the LED ON if the connection breaks (simulating wire failure).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON until switch is hit | Switch wired to NC instead of NO | Swap switch contacts from NC to NO, or invert the logic check in the code (`if (limitState == HIGH)`). |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [17 - Pico Button Pull-up](17-pico-button-pullup.md)
- [29 - Pico Vibration Sensor](29-pico-vibration-sensor.md)
