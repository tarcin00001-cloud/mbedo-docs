# 21 - ESP32 TTP223 Touch Lamp

Use a capacitive touch sensor (TTP223 module) to control an LED lamp — touching the pad turns the light on, releasing it turns it off.

## Goal
Learn how a capacitive touch sensor module produces a digital HIGH/LOW signal and how to read it the same way as a regular push button.

## What You Will Build
A TTP223 capacitive touch module connected to GPIO 4. Touching the conductive pad drives the output HIGH and lights an LED on GPIO 5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| TTP223 Touch Sensor Module | `touch_sensor` | Yes | Yes |
| LED (any colour) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| TTP223 Module | VCC | 3V3 | Red | Module supply voltage |
| TTP223 Module | GND | GND | Black | Module ground |
| TTP223 Module | SIG (OUT) | GPIO4 | Yellow | Digital output — HIGH on touch |
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Lamp output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The TTP223 module contains its own internal circuitry and requires no external pull resistors. Its SIG pin outputs a clean HIGH when touched and LOW when released. Keep signal wires short (under 20 cm) to reduce stray capacitance that could cause false triggers.

## Code
```cpp
// TTP223 Touch Lamp
const int TOUCH_PIN = 4;   // TTP223 SIG output
const int LED_PIN   = 5;   // Lamp LED

void setup() {
  pinMode(TOUCH_PIN, INPUT);    // TTP223 drives this pin — no pull needed
  pinMode(LED_PIN,   OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("TTP223 Touch Lamp ready — touch the pad.");
}

void loop() {
  int touched = digitalRead(TOUCH_PIN);
  digitalWrite(LED_PIN, touched);

  if (touched) {
    Serial.println("TOUCH detected — Lamp ON");
  } else {
    Serial.println("No touch  — Lamp OFF");
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Touch Sensor (TTP223)**, and **LED** onto the canvas.
2. Connect Touch Sensor **SIG** to **GPIO4**.
3. Connect LED **anode** to **GPIO5** and **cathode** to **GND**.
4. Paste the code and click **Run**.
5. Click the touch sensor widget on the canvas to simulate a touch event.

## Expected Output
Serial Monitor:
```
TTP223 Touch Lamp ready — touch the pad.
No touch  — Lamp OFF
TOUCH detected — Lamp ON
TOUCH detected — Lamp ON
No touch  — Lamp OFF
```

## Expected Canvas Behavior
* The LED component stays off while the touch sensor widget is idle.
* Clicking the touch sensor widget illuminates the LED component immediately.
* Releasing the widget extinguishes the LED.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(TOUCH_PIN, INPUT)` | Configures GPIO 4 as a plain input — the TTP223 drives the pin actively. |
| `int touched = digitalRead(TOUCH_PIN)` | Reads the TTP223 output: `1` when the pad is touched, `0` otherwise. |
| `digitalWrite(LED_PIN, touched)` | Mirrors the touch state directly to the LED without requiring an `if`. |
| `delay(100)` | Polling interval; short enough for a responsive lamp feel. |

## Hardware & Safety Concept: Capacitive Touch Sensing
A capacitive touch sensor detects the small change in electrical capacitance caused by a human finger approaching or touching a conductive pad. The human body acts as a grounded conductor; bringing a finger near the pad slightly increases the capacitance between the pad and ground. The TTP223 chip measures this change with an internal oscillator — when the oscillation frequency shifts beyond a calibrated threshold, the chip asserts its output HIGH. Unlike mechanical buttons, capacitive sensors have **no moving parts**, resulting in near-unlimited operational life. They are used in smartphones, household appliances, and industrial panels for this reason.

## Try This! (Challenges)
1. **Touch counter**: Count how many times the pad is touched and print the count.
2. **Timed lamp**: Turn the LED on for exactly 5 seconds after each touch, then automatically turn off.
3. **Sensitivity test**: Bring your finger progressively closer without touching — observe at what distance the sensor triggers.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor triggers without touch | Stray capacitance from long wires | Shorten SIG wire; keep away from power traces |
| Sensor never triggers | SIG wire not connected | Verify GPIO4 is wired to TTP223 SIG pin |
| LED always on | TTP223 VCC/GND swapped | Check module orientation and power pins |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [22 - ESP32 Touch Toggle Controller](22-esp32-touch-toggle-controller.md)
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
