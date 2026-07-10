# 22 - ESP32 Touch Toggle Controller

Use a TTP223 capacitive touch sensor to **toggle** an LED on and off with each tap — one touch turns the light on; the next touch turns it off.

## Goal
Learn how to implement a software latch (toggle flip-flop) using a single sensor input, so that each distinct touch event flips the output state rather than mirroring the sensor level.

## What You Will Build
A TTP223 touch module on GPIO 4 and an LED on GPIO 5. Each time the pad is tapped and released, the LED flips between on and off — behaving like a touch-sensitive light switch.

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
| LED | Anode (+) | GPIO5 via 330 Ω | Orange | Toggle output |
| LED | Cathode (−) | GND | Black | Ground return |

> **Wiring tip:** The TTP223 module's SIG pin goes HIGH while your finger is on the pad and LOW when you lift it. The toggle logic watches for the HIGH-to-LOW transition (finger lift) to fire the toggle — this prevents the state from flipping repeatedly while the finger remains on the pad.

## Code
```cpp
// Touch Toggle Controller — tap once ON, tap again OFF
const int TOUCH_PIN = 4;
const int LED_PIN   = 5;

bool ledState    = false;   // Current lamp state
bool lastTouched = false;   // Previous sensor state for edge detection

void setup() {
  pinMode(TOUCH_PIN, INPUT);
  pinMode(LED_PIN,   OUTPUT);
  digitalWrite(LED_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Touch Toggle Controller ready.");
}

void loop() {
  bool touched = digitalRead(TOUCH_PIN);

  // Detect the rising edge: sensor just went from NOT-touched to TOUCHED
  if (touched && !lastTouched) {
    ledState = !ledState;                  // Flip the lamp state
    digitalWrite(LED_PIN, ledState);
    Serial.print("Toggle! LED is now: ");
    Serial.println(ledState ? "ON" : "OFF");
  }

  lastTouched = touched;   // Remember current state for next loop
  delay(50);               // Debounce interval
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Touch Sensor (TTP223)**, and **LED** onto the canvas.
2. Connect Touch Sensor **SIG** to **GPIO4**.
3. Connect LED **anode** to **GPIO5** and **cathode** to **GND**.
4. Paste the code and click **Run**.
5. Click the touch sensor widget repeatedly — the LED should flip state on each tap.

## Expected Output
Serial Monitor:
```
Touch Toggle Controller ready.
Toggle! LED is now: ON
Toggle! LED is now: OFF
Toggle! LED is now: ON
Toggle! LED is now: OFF
```

## Expected Canvas Behavior
* First tap: LED turns on and stays on.
* Second tap: LED turns off and stays off.
* Each subsequent tap continues to alternate between on and off.
* No Serial output is printed while holding — only on each fresh touch.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool lastTouched = false` | Stores the sensor reading from the previous loop iteration. |
| `if (touched && !lastTouched)` | Detects a **rising edge** — sensor just became HIGH (finger just landed). |
| `ledState = !ledState` | Inverts the boolean lamp state (C++ NOT operator). |
| `lastTouched = touched` | Updates the memory variable at the end of every loop for the next comparison. |
| `delay(50)` | 50 ms debounce prevents the capacitive sensor's own charge stabilisation from triggering multiple edges. |

## Hardware & Safety Concept: Edge Detection and Latching
A **latch** is a circuit or software pattern that holds a state even after the input that caused it has disappeared. Without a latch, a sensor that goes HIGH while held would produce output only while held — like Project 21. With a latch, a brief touch causes a **permanent state change** until the next touch resets it. Edge detection is the technique of acting only on a signal transition (LOW→HIGH or HIGH→LOW) rather than the steady level. It is the foundation of event-driven programming used in all modern UIs, IoT devices, and industrial PLCs.

## Try This! (Challenges)
1. **Two-LED toggle**: Toggle LED A on the first touch, LED B on the second touch, and back to idle on the third.
2. **Touch-count display**: Count toggles and print the total every 5 taps.
3. **Auto-off timer**: After toggling ON, start a 10-second timer and automatically turn the LED off if not touched again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED toggles multiple times per tap | Debounce too short or stray capacitance | Increase `delay(50)` to `delay(150)` |
| Toggle never fires | Rising-edge condition fails | Check `lastTouched` is initialised to `false` and updates each loop |
| LED flickers during hold | Level detection instead of edge detection | Confirm the `if` checks `touched && !lastTouched`, not just `touched` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [21 - ESP32 TTP223 Touch Lamp](21-esp32-ttp223-touch-lamp.md)
- [26 - ESP32 Button Press Counter](26-esp32-button-press-counter.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
