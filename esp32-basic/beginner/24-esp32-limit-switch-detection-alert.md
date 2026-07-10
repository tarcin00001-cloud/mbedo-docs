# 24 - ESP32 Limit Switch Detection Alert

Detect when a mechanical limit switch is physically actuated and trigger an audible and visual alert on the ESP32.

## Goal
Learn how to wire a normally-open (NO) limit switch with an internal pull-up resistor and generate a multi-output alert when the switch closes.

## What You Will Build
A limit switch on GPIO 4 (using `INPUT_PULLUP`) drives an LED on GPIO 5 and an active buzzer on GPIO 15 whenever the switch lever is pressed down.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Limit Switch (NO type) | `switch` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Limit Switch | Common (C) | GND | Black | Common connects to ground |
| Limit Switch | Normally Open (NO) | GPIO4 | Yellow | Signal — pulled LOW when actuated |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Alert LED |
| LED | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Buzzer control signal |
| Active Buzzer | GND (−) | GND | Black | Ground return |

> **Wiring tip:** `INPUT_PULLUP` on GPIO 4 holds the pin at HIGH (via the internal ~45 kΩ resistor). When the limit switch lever is pressed, the NO contact closes and connects GPIO 4 directly to GND, pulling it LOW. This active-LOW detection is the industry standard for limit switches because a broken wire also reads LOW, triggering the alert rather than silently failing.

## Code
```cpp
// Limit Switch Detection Alert
const int LIMIT_PIN  = 4;    // Limit switch NO contact
const int LED_PIN    = 5;    // Alert LED
const int BUZZER_PIN = 15;   // Active buzzer

void setup() {
  pinMode(LIMIT_PIN,  INPUT_PULLUP); // Internal pull-up — active LOW
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Limit Switch Alert ready — waiting for actuation.");
}

void loop() {
  int limitState = digitalRead(LIMIT_PIN);

  if (limitState == LOW) {
    // Limit switch actuated — lever pressed
    digitalWrite(LED_PIN,    HIGH);
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println("!! LIMIT SWITCH ACTUATED — Position reached !!");
  } else {
    // Switch open — normal idle state
    digitalWrite(LED_PIN,    LOW);
    digitalWrite(BUZZER_PIN, LOW);
    Serial.println("Limit switch: IDLE");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Limit Switch**, **LED**, and **Buzzer** onto the canvas.
2. Connect Limit Switch **signal** to **GPIO4**.
3. Connect LED **anode** to **GPIO5**, Buzzer **VCC** to **GPIO15**.
4. Paste the code and click **Run**.
5. Click the limit switch widget to simulate the lever being pressed.

## Expected Output
Serial Monitor:
```
Limit Switch Alert ready — waiting for actuation.
Limit switch: IDLE
Limit switch: IDLE
!! LIMIT SWITCH ACTUATED — Position reached !!
!! LIMIT SWITCH ACTUATED — Position reached !!
Limit switch: IDLE
```

## Expected Canvas Behavior
* LED and buzzer remain off during normal idle state.
* Clicking the limit switch widget immediately lights the LED and activates the buzzer.
* Both alerts extinguish as soon as the switch is released.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(LIMIT_PIN, INPUT_PULLUP)` | Enables internal pull-up — pin is HIGH at rest, goes LOW when switch closes. |
| `if (limitState == LOW)` | Active-LOW detection: triggers the alert when the switch contact closes to GND. |
| `digitalWrite(LED_PIN, HIGH)` | Lights the alert LED during actuation. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Activates the buzzer simultaneously for an audio-visual alert. |

## Hardware & Safety Concept: Limit Switches in Machinery
Limit switches are electro-mechanical sensors used in industrial machines, 3D printers, CNC routers, and elevators to detect when a moving part has reached a physical boundary. They are wired in **normally-open (NO)** configuration with a pull-up resistor. When the lever is struck by the moving part, the contacts close and the system knows the boundary has been reached. The machine can then stop, reverse, or trigger an alarm. The active-LOW detection standard (NO contact to GND with pull-up) provides **fail-safe** behaviour: if the wire breaks or the switch fails open, the controller still reads LOW and can halt the machine.

## Try This! (Challenges)
1. **Latching alarm**: Once actuated, keep the buzzer on until a separate reset button is pressed.
2. **Actuation counter**: Count how many times the switch is actuated during a session and log the total every 10 actuations.
3. **Buzzer pattern**: Replace the continuous buzzer tone with 3 short beeps when actuated.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alert triggers continuously with no switch press | Switch Common wired to 3V3 instead of GND | Move the Common wire to GND |
| No alert when switch pressed | Switch NO pin not connected to GPIO 4 | Re-seat the NO-to-GPIO4 wire |
| Buzzer buzzes faintly | Buzzer wired backwards | Check buzzer polarity — positive leg to GPIO 15 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [23 - ESP32 Slide Switch State Logging](23-esp32-slide-switch-state-logging.md)
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
