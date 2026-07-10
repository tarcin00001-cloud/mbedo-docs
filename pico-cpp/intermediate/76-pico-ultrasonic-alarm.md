# 76 - Pico Ultrasonic Alarm

Build a proximity alarm that plays an acoustic warning if an object gets too close.

## Goal
Learn how to map distance readings to active buzzer control logic to create safety zones.

## What You Will Build
A backup collision warning sensor:
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Monitors distance.
- **Active Buzzer (GP10)**: Sounds a rapid pulsing alarm if the distance drops below 30 cm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider resistors) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | Trig | GP14 | Trigger signal |
| HC-SR04 | Echo | GP15 | Echo signal (requires 5V to 3.3V divider) |
| HC-SR04 | GND | GND | Ground reference |
| Active Buzzer | VCC (+) | GP10 | Buzzer control |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int TRIG_PIN   = 14;
const int ECHO_PIN   = 15;
const int BUZZER_PIN = 10;

const int SAFE_DISTANCE = 30; // Limit in cm

void setup() {
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  // Generate trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Trigger alert if boundary is breached
  if (distance > 0 && distance < SAFE_DISTANCE) {
    // Pulse alarm rapidly
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    digitalWrite(BUZZER_PIN, LOW); // Safe, silence
  }

  delay(300);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, and **Active Buzzer** onto the canvas.
2. Connect HC-SR04: **Trig** to **GP14**, **Echo** to **GP15** (with voltage divider), **VCC** to **5V**, **GND** to **GND**.
3. Connect Buzzer: **VCC** to **GP10**, **GND** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Move the HC-SR04 target obstacle closer than 30 cm and listen to the rapid alarm beep.

## Expected Output

Terminal:
```
Simulation active. Proximity alert active. Boundary: 30cm.
```

## Expected Canvas Behavior
| Target Distance | GP10 (Buzzer) State | Status |
| --- | --- | --- |
| > 30 cm | LOW | Safe |
| < 30 cm | Pulsing HIGH/LOW | **COLLISION DANGER WARNING** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `distance > 0 && distance < SAFE_DISTANCE` | Logical check ignoring 0 readings (timeout errors) while checking if the proximity limit is breached. |

## Hardware & Safety Concept: Sensor Blind Zones
Ultrasonic sensors measure distances by sending sound waves and timing the reflection return. However, they have a **minimum blind zone** (typically 2 cm to 3 cm). If an object is closer than 2 cm, the echo returns before the receiver has finished switching from transmission mode, resulting in invalid or zero readings. Always filter out 0 readings in safety logic.

## Try This! (Challenges)
1. **Dynamic Alert Rate**: Change the warning beep speed dynamically based on distance (e.g. beeping faster as the target gets closer).
2. **Alert LED**: Add a Red LED on GP13 that flashes in sync with the buzzer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers continuously when clear | Invalid 0 readings | Ensure your trigger logic checks `distance > 0` to filter out echo timeout errors. |

## Mode Notes
This basic proximity sensor project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [75 - Pico Ultrasonic Serial](75-pico-ultrasonic-serial.md)
- [77 - Pico Ultrasonic LCD](77-pico-ultrasonic-lcd.md)
