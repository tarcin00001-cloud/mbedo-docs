# 03 - ESP32 Alternate LED Blink

Build a dual-channel railway crossing signal warning indicator that toggles two external LEDs alternately.

## Goal
Learn how to control multiple GPIO outputs, synchronize opposite logic states (HIGH/LOW), and write clean C++ digital control logic.

## What You Will Build
Two external LEDs connected to GPIO 22 and GPIO 23 toggle states alternately (when LED A is ON, LED B is OFF; then LED A is OFF, LED B is ON) every 500 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED × 2 (e.g., Red & Blue) | `led` | Yes (two LEDs) | Yes |
| 220 Ω Resistors × 2 | `resistor` | Optional | Yes (current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED A (Red) | Anode (via 220 Ω) | GPIO22 | Red | Toggled output channel A |
| LED A (Red) | Cathode | GND | Black | Ground return |
| LED B (Blue) | Anode (via 220 Ω) | GPIO23 | Blue | Toggled output channel B |
| LED B (Blue) | Cathode | GND | Black | Ground return |

> **Wiring tip:** Both LEDs require separate current-limiting resistors. Connect LED A to GPIO 22 and LED B to GPIO 23. Both cathodes connect to a shared ground rail.

## Code
```cpp
// Define external LED pins
const int LED_A_PIN = 22;
const int LED_B_PIN = 23;

const int TOGGLE_DELAY = 500; // Alternate delay in milliseconds

void setup() {
  pinMode(LED_A_PIN, OUTPUT);
  pinMode(LED_B_PIN, OUTPUT);
  
  Serial.begin(115200);
  Serial.println("ESP32 Alternate Blinker Active.");
}

void loop() {
  // State 1: LED A ON, LED B OFF
  digitalWrite(LED_A_PIN, HIGH);
  digitalWrite(LED_B_PIN, LOW);
  Serial.println("State: A=ON, B=OFF");
  delay(TOGGLE_DELAY);
  
  // State 2: LED A OFF, LED B ON
  digitalWrite(LED_A_PIN, LOW);
  digitalWrite(LED_B_PIN, HIGH);
  Serial.println("State: A=OFF, B=ON");
  delay(TOGGLE_DELAY);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **two LEDs** onto the canvas.
2. Connect LED 1 **A** to **GPIO22**, and LED 2 **A** to **GPIO23**.
3. Connect both LED **C** pins to ESP32 **GND**.
4. Paste code, select interpreted mode, and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Alternate Blinker Active.
State: A=ON, B=OFF
State: A=OFF, B=ON
State: A=ON, B=OFF
```

## Expected Canvas Behavior
* LED A (Red) and LED B (Blue) alternate flashing.
* The state outputs are logged to the Serial Monitor every 500 ms.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(LED_A_PIN, OUTPUT);` | Configures GPIO 22 as output. |
| `pinMode(LED_B_PIN, OUTPUT);` | Configures GPIO 23 as output. |
| `digitalWrite(LED_A_PIN, HIGH);` | Turns on LED A while LED B is turned off. |

## Hardware & Safety Concept: Pin Sourcing and Sinking Limits
Microcontrollers have two limits for current:
1. **Per-pin current limit**: For ESP32, this is 20 mA.
2. **Total package current limit**: The sum of current through all GPIO pins simultaneously must not exceed **80 mA**.
If you connect four LEDs drawing 20 mA each, you hit the package limit. Alternating two LEDs ensures that the total active current draw remains constant and safe, preventing thermal stress on the CPU chip.

## Try This! (Challenges)
1. **Dual Strobe Flash**: Modify the code to make LED A flash twice quickly, then LED B flash twice quickly, creating a police cruiser warning pattern.
2. **Duty Cycle Modifier**: Create a variable delay that speeds up the switching rate over 10 seconds before slowing back down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One LED works, the other stays off | Pin number mismatch | Check your wiring. Ensure the non-working LED is connected to the exact pin specified in the code (GPIO23). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - ESP32 LED Blink Interval](02-esp32-led-blink-interval.md)
- [04 - ESP32 Multiple LED Chase](04-esp32-multiple-led-chase.md)
