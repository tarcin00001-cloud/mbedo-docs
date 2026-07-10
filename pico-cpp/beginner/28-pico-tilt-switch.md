# 28 - Pico Tilt Switch

Read a tilt switch sensor to detect mechanical inclination changes.

## Goal
Learn how to interface tilt-ball sensors with digital input pins on the Raspberry Pi Pico to detect orientation shifts.

## What You Will Build
An inclination level indicator:
- **Tilt Switch (GP16)**: Monitored via internal pull-up.
- **Normal LED (GP14)**: Glows Green when the sensor is level (switch open / HIGH).
- **Tilt LED (GP15)**: Glows Red when the sensor is tilted (switch closed / LOW).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Tilt Sensor (Ball-type) | `button` | Yes (represented by switch button) | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Optional | Yes (two resistors) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Tilt Switch | Terminal 1 | GP16 | Sense pin |
| Tilt Switch | Terminal 2 | GND | Ground return |
| Green LED | Anode | GP14 | Level indicator |
| Green LED | Cathode | GND | Ground return via resistor |
| Red LED | Anode | GP15 | Tilt alarm indicator |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int TILT_PIN  = 16;
const int NORMAL_LED = 14;
const int ALARM_LED  = 15;

void setup() {
  pinMode(TILT_PIN, INPUT_PULLUP);
  pinMode(NORMAL_LED, OUTPUT);
  pinMode(ALARM_LED, OUTPUT);
}

void loop() {
  int tiltState = digitalRead(TILT_PIN);

  // Active-LOW check: switch closes (LOW) when tilted
  if (tiltState == LOW) {
    digitalWrite(ALARM_LED, HIGH);  // Tilted alert ON (Red)
    digitalWrite(NORMAL_LED, LOW);  // Normal OFF
  } else {
    digitalWrite(ALARM_LED, LOW);   // Tilted alert OFF
    digitalWrite(NORMAL_LED, HIGH); // Normal ON (Green)
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Tilt Switch** (represented by switch button), **Green LED**, and **Red LED** onto the canvas.
2. Connect Switch: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect Green LED: **Anode** to **GP14**, **Cathode** to **GND**.
4. Connect Red LED: **Anode** to **GP15**, **Cathode** to **GND**.
5. Paste code, select the interpreted mode, and click **Run**.
6. Simulate a tilt by closing the switch contact on canvas.

## Expected Output

Terminal:
```
Simulation active. Inclination monitor active.
```

## Expected Canvas Behavior
| Switch state (GP16) | GP14 (Green LED) | GP15 (Red LED) | Orientation |
| --- | --- | --- | --- |
| open (HIGH) | HIGH (ON) | LOW (OFF) | Level / Safe |
| Closed (LOW) | LOW (OFF) | HIGH (ON) | **Tilted / Alert** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(NORMAL_LED, HIGH)` | Drives GP14 HIGH to indicate a level orientation. |

## Hardware & Safety Concept: Metallic Ball Tilt Sensors
A basic tilt switch contains a tiny metallic ball inside a non-conductive tube with two electrodes at the bottom. When the sensor is tilted downward, the ball rolls onto the electrodes, completing the circuit. Because the ball can bounce slightly during sudden movements, software delays are needed to filter out false tilt logs.

## Try This! (Challenges)
1. **Tilt Siren**: Add a buzzer on GP10 and sound a warning tone only when tilted.
2. **Latch Alarm**: Modify the code so that if the sensor is tilted once, the Red LED stays ON forever until the Pico reset button is clicked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Red LED flickers when level | Sensor is oriented incorrectly | Mechanical tilt switches must be mounted in the correct physical direction to ensure the ball rolls away from contacts when flat. |

## Mode Notes
This basic GPIO project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [17 - Pico Button Pull-up](17-pico-button-pullup.md)
- [24 - Pico Limit Switch](24-pico-limit-switch.md)
