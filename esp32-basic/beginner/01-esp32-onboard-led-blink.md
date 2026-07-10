# 01 - ESP32 Onboard LED Blink

Build your first ESP32 program by blinking the built-in LED at a steady one-second interval.

## Goal
Learn how to control the ESP32's onboard LED using standard Arduino functions, understand the default GPIO mapping, and write a basic `setup()` and `loop()` program structure.

## What You Will Build
The built-in LED on the ESP32 Development Board, which is hardwired to GPIO 2, flashes ON for one second and OFF for one second, repeating indefinitely.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Onboard LED | `led` | Yes (built-in) | Yes (built-in) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED | Anode | GPIO2 | Internal | Hardwired internally on the ESP32 DevKit |
| Onboard LED | Cathode | GND | Internal | Internal ground return path |

> **Wiring tip:** The onboard LED is pre-wired to GPIO 2 on most ESP32 DevKit boards. No external wiring is needed to test this project.

## Code
```cpp
// Onboard LED is connected to GPIO 2 on the ESP32 DevKitC
const int LED_PIN = 2;

void setup() {
  // Configure the LED pin as an output
  pinMode(LED_PIN, OUTPUT);
  
  // Initialize Serial communication at 115200 baud
  Serial.begin(115200);
  Serial.println("ESP32 Onboard LED Blink Initialised.");
}

void loop() {
  // Turn the LED ON (HIGH voltage level)
  digitalWrite(LED_PIN, HIGH);
  Serial.println("LED State: ON");
  delay(1000); // Wait for 1 second (1000 milliseconds)
  
  // Turn the LED OFF (LOW voltage level)
  digitalWrite(LED_PIN, LOW);
  Serial.println("LED State: OFF");
  delay(1000); // Wait for 1 second (1000 milliseconds)
}
```

## What to Click in MbedO
1. Drag the **ESP32 DevKitC** onto the canvas.
2. No external connections are required.
3. Paste the code into the editor.
4. Select the default interpreted C++ runtime.
5. Click **Run**.
6. Observe the green onboard LED near the USB port blink, and check the Serial Monitor logs.

## Expected Output
Serial Monitor:
```
ESP32 Onboard LED Blink Initialised.
LED State: ON
LED State: OFF
LED State: ON
```

## Expected Canvas Behavior
* On startup, the Serial Monitor prints the initialization message.
* Every 1 second, the onboard LED (labelled `LED` or `D2` on the ESP32 chip body) switches between bright green and black (OFF).
* The console prints the corresponding state message synchronously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int LED_PIN = 2;` | Assigns the identifier `LED_PIN` to GPIO 2, which matches the hardwired LED on the DevKitC. |
| `pinMode(LED_PIN, OUTPUT);` | Configures GPIO 2 to act as a digital output, allowing it to supply voltage to the LED. |
| `Serial.begin(115200);` | Opens a serial interface at 115200 bits per second (standard baud rate for ESP32 boards). |
| `digitalWrite(LED_PIN, HIGH);` | Drives GPIO 2 to 3.3V, lighting the internal LED. |

## Hardware & Safety Concept: ESP32 Logic Levels
Unlike the 5V logic levels of the Arduino Uno, the ESP32 operates at a **3.3V logic level**. Pushing more than 3.3V into any input pin or drawing too much current (above **20 mA** per pin) will destroy the silicon gates. The onboard LED includes a surface-mount current-limiting resistor (typically 1k-ohm) to keep current draw safely around 1–2 mA.

## Try This! (Challenges)
1. **Heartbeat Double Blink**: Modify the delay parameters to make the LED blink twice in rapid succession (80 ms on, 80 ms off, 80 ms on), followed by a 1-second pause.
2. **Speed Sweep**: Write a loop that decreases the delay time by 100 ms on each cycle down to a minimum of 100 ms, then resets.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Onboard LED does not blink | Incorrect pin assignment | Some ESP32 clones use GPIO 4 or GPIO 12 for the onboard LED. Change `LED_PIN` value to test other common configurations. |
| Serial Monitor prints garbled characters | Baud rate mismatch | Ensure the Serial Monitor window baud rate dropdown is set to **115200** to match the sketch configuration. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - ESP32 LED Blink Interval](02-esp32-led-blink-interval.md)
- [03 - ESP32 Alternate LED Blink](03-esp32-alternate-led-blink.md)
