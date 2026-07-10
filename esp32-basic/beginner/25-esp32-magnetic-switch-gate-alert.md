# 25 - ESP32 Magnetic Switch Gate Alert

Use a reed switch (magnetic contact sensor) to detect whether a door or gate is open or closed and print an alert when it opens.

## Goal
Learn how a reed switch outputs a digital signal in the presence or absence of a magnet, and how to log gate status changes to the Serial Monitor.

## What You Will Build
A magnetic reed switch wired to GPIO 4 with `INPUT_PULLUP`. When the magnet (attached to a door/gate) is close to the switch, the contacts close and the pin reads LOW. When the door opens and the magnet moves away, the contacts open, the pull-up drives the pin HIGH, and a "GATE OPEN" alert fires.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Reed Switch (magnetic contact sensor) | `switch` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Reed Switch | Pin 1 | GPIO4 | Yellow | Signal input — active-LOW when magnet present |
| Reed Switch | Pin 2 | GND | Black | Ground — contacts close to GND when activated |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Alert indicator |
| LED | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Buzzer control |
| Active Buzzer | GND (−) | GND | Black | Ground return |

> **Wiring tip:** A reed switch has only two leads and is not polarised — either lead can go to GPIO 4 or GND. The `INPUT_PULLUP` mode internally holds GPIO 4 HIGH. When the reed contacts close (magnet nearby), GPIO 4 is shorted to GND and reads LOW. When the magnet is removed (gate open), the contacts open, the pull-up restores HIGH, and the alert triggers.

## Code
```cpp
// Magnetic Switch Gate Alert
const int REED_PIN   = 4;    // Reed switch input
const int LED_PIN    = 5;    // Alert LED
const int BUZZER_PIN = 15;   // Alert buzzer

bool lastGateOpen = false;   // Track previous state for edge detection

void setup() {
  pinMode(REED_PIN,   INPUT_PULLUP); // Internal pull-up — LOW = magnet present = gate closed
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Magnetic Gate Alert ready.");
}

void loop() {
  // LOW = magnet present = gate CLOSED
  // HIGH = magnet absent  = gate OPEN
  bool gateOpen = (digitalRead(REED_PIN) == HIGH);

  if (gateOpen && !lastGateOpen) {
    // Gate just opened — fire alert
    digitalWrite(LED_PIN,    HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println("!! GATE OPENED — ALERT !!");
  } else if (!gateOpen && lastGateOpen) {
    // Gate just closed — clear alert
    digitalWrite(LED_PIN,    LOW);
    digitalWrite(BUZZER_PIN, LOW);
    Serial.println("Gate closed — System secure.");
  }

  lastGateOpen = gateOpen;
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Reed Switch**, **LED**, and **Buzzer** onto the canvas.
2. Connect Reed Switch **signal pin** to **GPIO4**.
3. Connect LED **anode** to **GPIO5**, Buzzer **VCC** to **GPIO15**.
4. Paste the code and click **Run**.
5. Toggle the reed switch widget OFF (simulating gate open) — alert fires.
6. Toggle the reed switch widget ON (simulating gate closed) — alert clears.

## Expected Output
Serial Monitor:
```
Magnetic Gate Alert ready.
Gate closed — System secure.
!! GATE OPENED — ALERT !!
Gate closed — System secure.
!! GATE OPENED — ALERT !!
```

## Expected Canvas Behavior
* LED and buzzer are off when the reed switch is closed (magnet present).
* LED lights and buzzer sounds the moment the switch opens (gate opened).
* Alert clears immediately when the switch closes again (gate secured).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool gateOpen = (digitalRead(REED_PIN) == HIGH)` | Maps the inverted active-LOW signal to an intuitive boolean: `true` = gate open. |
| `if (gateOpen && !lastGateOpen)` | Rising-edge detection: fires only when gate transitions from closed to open. |
| `else if (!gateOpen && lastGateOpen)` | Falling-edge detection: clears the alert when gate transitions from open to closed. |
| `lastGateOpen = gateOpen` | Saves current state for the next loop comparison. |

## Hardware & Safety Concept: Reed Switches and Magnetic Contact Sensors
A reed switch is a glass-enclosed pair of thin ferromagnetic reeds (contacts) that magnetically attract and close when a permanent magnet is placed nearby. When the magnet is removed, the spring tension in the reeds pushes them apart, opening the circuit. Reed switches have **no electrical contact with the magnet** — they are contactless when the magnet is far, and contact-making when it is near. This makes them ideal for door and window alarms: the switch body mounts on the frame and the magnet mounts on the moving door. They are also completely sealed in glass, making them weather-resistant and suitable for outdoor use. Modern security alarm systems use the same fundamental component.

## Try This! (Challenges)
1. **Open duration timer**: Record the time the gate opened with `millis()` and print how long it stayed open when it closes.
2. **Intrusion count**: Increment a counter on every gate-open event and print total breaches.
3. **Multi-gate system**: Add a second reed switch on GPIO 18 to monitor two gates independently.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alert fires continuously with magnet present | Reed switch pins swapped or pull-up direction wrong | Ensure one pin to GPIO 4 and other to GND; confirm `INPUT_PULLUP` in code |
| No alert when magnet removed | Wrong GPIO number | Verify reed switch lead is on GPIO 4, not a neighbouring pin |
| Alert briefly fires when magnet applied | Mechanical reed bounce | Increase the `delay(100)` polling interval to 200 ms |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
