# 30 - ESP32 Laser Receiver Signal Detector

Use a KY-008 laser transmitter and a laser receiver (photoresistor-based detector) module to detect beam interruption and trigger an alert.

## Goal
Learn how a laser beam-break circuit works, how to read a digital detector module, and how to trigger a multi-output alert when the laser path is interrupted.

## What You Will Build
A KY-008 laser module aimed at a laser receiver detector module. The detector output is connected to GPIO 4. When the beam is broken (someone or something crosses the path), a red LED on GPIO 5 lights and a buzzer on GPIO 15 sounds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| KY-008 Laser Transmitter Module | `laser` | Yes | Yes |
| Laser Receiver / Detector Module | `laser_receiver` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| KY-008 Laser Module | VCC (middle pin) | 3V3 | Red | Laser diode supply |
| KY-008 Laser Module | GND (−) | GND | Black | Laser module ground |
| KY-008 Laser Module | Signal (S) | Not connected | — | Signal pin unused for simple on/off |
| Laser Receiver Module | VCC | 3V3 | Red | Detector module supply |
| Laser Receiver Module | GND | GND | Black | Detector module ground |
| Laser Receiver Module | DO (digital out) | GPIO4 | Yellow | HIGH when beam detected, LOW when broken |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Beam-break alert |
| LED (red) | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alert buzzer |
| Active Buzzer | GND (−) | GND | Black | Ground return |

> **Wiring tip:** Align the laser dot precisely onto the receiver sensor surface — even a few millimetres off-centre can prevent the module from seeing the beam. Power the laser from 3V3 (not 5V) when used with an ESP32 to avoid any voltage mismatch risks. The receiver DO pin goes HIGH when the beam is received (threshold met) and LOW when the beam is blocked. This active-LOW alert logic means the alarm fires on LOW.

## Code
```cpp
// Laser Beam-Break Signal Detector
const int LASER_RX_PIN = 4;    // Receiver digital output
const int LED_PIN      = 5;    // Alert LED
const int BUZZER_PIN   = 15;   // Alert buzzer

void setup() {
  pinMode(LASER_RX_PIN, INPUT);   // Detector module drives this pin
  pinMode(LED_PIN,      OUTPUT);
  pinMode(BUZZER_PIN,   OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Laser Beam Detector ready — beam active.");
}

void loop() {
  int beamState = digitalRead(LASER_RX_PIN);

  if (beamState == LOW) {
    // Beam is BROKEN — trigger alert
    digitalWrite(LED_PIN,    HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println("!! BEAM BROKEN — INTRUDER ALERT !!");
  } else {
    // Beam is CLEAR — system secure
    digitalWrite(LED_PIN,    LOW);
    digitalWrite(BUZZER_PIN, LOW);
    Serial.println("Beam: CLEAR");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Laser Transmitter**, **Laser Receiver**, **LED**, and **Buzzer** onto the canvas.
2. Connect Laser Receiver **DO** to **GPIO4**.
3. Connect LED **anode** to **GPIO5**, Buzzer **VCC** to **GPIO15**.
4. Paste the code and click **Run**.
5. In the canvas, toggle the laser receiver widget to simulate the beam being blocked.
6. Observe the LED and buzzer activate when the beam is broken.

## Expected Output
Serial Monitor:
```
Laser Beam Detector ready — beam active.
Beam: CLEAR
Beam: CLEAR
!! BEAM BROKEN — INTRUDER ALERT !!
!! BEAM BROKEN — INTRUDER ALERT !!
Beam: CLEAR
```

## Expected Canvas Behavior
* LED and buzzer are off when the beam is intact (receiver DO is HIGH).
* Toggling the receiver widget to blocked state immediately lights the LED and sounds the buzzer.
* Restoring the beam clears both alerts within 100 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(LASER_RX_PIN, INPUT)` | Configures GPIO 4 as a plain input — the receiver module actively drives the pin. |
| `if (beamState == LOW)` | Active-LOW detection: beam broken → DO goes LOW → alert fires. |
| `digitalWrite(LED_PIN, HIGH)` | Lights the red alert LED when the beam is broken. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Sounds the buzzer simultaneously with the LED. |
| `delay(100)` | 10 Hz polling rate — fast enough to catch a quick beam interruption reliably. |

## Hardware & Safety Concept: Laser Beam-Break Sensors
A laser beam-break detector is a type of **through-beam** optical sensor. The transmitter emits a focused beam; the receiver is positioned opposite and monitors whether the beam is present. Any opaque object crossing the path breaks the beam and triggers the output. This principle is used in: **access control barriers** (turnstiles, garage door safety beams), **production line part counters** (one count per interruption), **security tripwires**, and **speed traps** (two beams separated by a known distance, velocity = distance ÷ time between breaks). Laser-based detectors have much longer range than IR proximity sensors because the coherent beam does not diverge over distance.

## Try This! (Challenges)
1. **Object counter**: Count every LOW transition (beam break) and print the total — simulating a production-line part counter.
2. **Latching alarm**: Once the beam is broken, latch the alarm and require a separate reset button to clear it.
3. **Speed trap**: Add a second receiver on GPIO 18, separated 10 cm from the first. Record the `millis()` timestamp of each beam break and compute the speed of an object passing through.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alert always active (beam never detected) | Laser not aimed at receiver | Adjust laser and receiver alignment in a dark room to see the dot clearly |
| No response when beam blocked | Receiver DO wired to wrong pin | Confirm receiver DO goes to GPIO 4, not GPIO 2 or 3 |
| Alert flickers with beam intact | Partial obstruction or vibration | Secure laser and receiver on a rigid mount; check for ambient IR interference |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
