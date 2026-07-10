# 28 - ESP32 Tilt Sensor Level Indicator

Use a ball-tilt switch (SW-520D or similar) to detect when a surface is no longer level and indicate the tilt state with LEDs.

## Goal
Learn how a tilt sensor module produces a digital signal based on orientation, and how to use two LEDs to provide a clear Level / Tilted visual indicator.

## What You Will Build
A tilt switch on GPIO 4 (using `INPUT_PULLUP`). A green LED on GPIO 5 lights when the surface is level (switch closed). A red LED on GPIO 15 lights when the surface is tilted (switch open).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Ball Tilt Switch (SW-520D) | `tilt_sensor` | Yes | Yes |
| LED (green) | `led` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| 330 Ω Resistor (×2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Tilt Switch | Pin 1 | GPIO4 | Yellow | Signal input with internal pull-up |
| Tilt Switch | Pin 2 | GND | Black | Ground — contacts close to GND when level |
| Green LED | Anode (+) | GPIO5 via 330 Ω | Green | LEVEL indicator |
| Green LED | Cathode (−) | GND | Black | Ground return |
| Red LED | Anode (+) | GPIO15 via 330 Ω | Red | TILTED indicator |
| Red LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The SW-520D tilt switch contains a small metal ball inside a conductive cylinder. When the sensor is upright the ball rests on the two contacts at the bottom, closing the circuit (LOW with `INPUT_PULLUP`). When tilted past ~30°, the ball rolls off the contacts, opening the circuit (HIGH with pull-up). The sensor is not polarised — either pin can be GPIO or GND.

## Code
```cpp
// Tilt Sensor Level Indicator
const int TILT_PIN  = 4;    // Ball tilt switch
const int LED_OK    = 5;    // Green — LEVEL
const int LED_TILT  = 15;   // Red   — TILTED

void setup() {
  pinMode(TILT_PIN, INPUT_PULLUP); // Internal pull-up — LOW=level, HIGH=tilted
  pinMode(LED_OK,   OUTPUT);
  pinMode(LED_TILT, OUTPUT);
  digitalWrite(LED_OK,   LOW);
  digitalWrite(LED_TILT, LOW);

  Serial.begin(115200);
  Serial.println("Tilt Level Indicator ready.");
}

void loop() {
  int tiltRead = digitalRead(TILT_PIN);

  if (tiltRead == LOW) {
    // Ball on contacts — surface is LEVEL
    digitalWrite(LED_OK,   HIGH);
    digitalWrite(LED_TILT, LOW);
    Serial.println("Status: LEVEL  (green)");
  } else {
    // Ball rolled off — surface is TILTED
    digitalWrite(LED_OK,   LOW);
    digitalWrite(LED_TILT, HIGH);
    Serial.println("Status: TILTED (red)");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Tilt Sensor**, and two **LED** components (green and red) onto the canvas.
2. Connect Tilt Sensor **signal** to **GPIO4**.
3. Connect green LED **anode** to **GPIO5**, red LED **anode** to **GPIO15**.
4. Paste the code and click **Run**.
5. Toggle the tilt sensor widget to simulate orientation change.

## Expected Output
Serial Monitor:
```
Tilt Level Indicator ready.
Status: LEVEL  (green)
Status: LEVEL  (green)
Status: TILTED (red)
Status: TILTED (red)
Status: LEVEL  (green)
```

## Expected Canvas Behavior
* Green LED on, red LED off when the tilt sensor widget is in the closed/level position.
* Red LED on, green LED off when the tilt sensor widget is toggled to the open/tilted position.
* Both LEDs never illuminate simultaneously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(TILT_PIN, INPUT_PULLUP)` | Enables internal pull-up; ball contact closure to GND reads LOW (level). |
| `if (tiltRead == LOW)` | LOW = contacts closed = ball inside = surface level → green LED on. |
| `else` | HIGH = contacts open = ball rolled away = surface tilted → red LED on. |
| `delay(200)` | 200 ms polling rate — fast enough for a responsive indicator without flicker. |

## Hardware & Safety Concept: Ball Tilt Switches
A ball tilt switch (also called a mercury-free tilt sensor) uses a conductive metal ball inside a cylindrical housing. Two pins emerge from one end of the cylinder. When the sensor is held with those pins at the bottom, gravity keeps the ball resting on both pins, completing the circuit. When the sensor tilts beyond a threshold angle (typically 30°–45°), the ball rolls away from the pins, breaking the circuit. This simple, passive mechanism is used in: **anti-theft luggage alarms** (tipping the bag breaks the circuit and triggers an alert), **automatic screen rotation** in older devices, **pinball machine tilt detectors**, and **appliance tip-over safety cutoffs** (electric heaters, kettles). Unlike MEMS accelerometers, ball switches have no electronics — they are purely mechanical.

## Try This! (Challenges)
1. **Buzzer alarm**: Add a buzzer on GPIO 25 that beeps when the red LED is on.
2. **Tilt duration log**: Measure how long the surface stays tilted with `millis()` and print it when it returns to level.
3. **Two-axis detection**: Add a second tilt sensor on GPIO 18 rotated 90° — detect tilt in two perpendicular planes independently.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both LEDs off, no response | Tilt switch pins not connected | Verify GPIO 4 lead and GND lead are in sensor legs |
| Red LED always on | Sensor mounted upside down | Flip the tilt sensor so the contact pins face downward |
| Flickering between states near threshold | Vibration near the tilt angle | Hold the surface more firmly or add a 50 ms debounce delay before switching |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
