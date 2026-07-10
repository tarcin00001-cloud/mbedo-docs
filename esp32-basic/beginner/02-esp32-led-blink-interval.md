# 02 - ESP32 LED Blink Interval

Build a warning flash beacon using an external LED and configure asymmetric blink intervals using custom delay parameters.

## Goal
Learn how to interface an external LED with the ESP32, wire a current-limiting resistor, and implement custom, asymmetric delay intervals (different ON/OFF durations) in C++.

## What You Will Build
An external LED connected to GPIO 22 flashes in a strobe pattern (ON for 200 ms, OFF for 1000 ms), creating an active warning beacon.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red or Yellow) | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Optional | Yes (current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (via 220 Ω) | GPIO22 | Red | Signal output pin |
| LED | Cathode | GND | Black | Ground return |

> **Wiring tip:** Connect the longer leg (Anode) of the LED to one side of the 220 ohm resistor, and the other side of the resistor to ESP32 pin GPIO22. Connect the shorter leg (Cathode) of the LED directly to a GND pin.

## Code
```cpp
// External LED connected to GPIO 22
const int LED_PIN = 22;

// Custom blink intervals in milliseconds
const int ON_TIME = 200;   // Short flash duration
const int OFF_TIME = 1000; // Long pause duration

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(115200);
  Serial.println("ESP32 Strobe Beacon Initialised.");
}

void loop() {
  // Flash ON
  digitalWrite(LED_PIN, HIGH);
  Serial.print("Strobe! (");
  Serial.print(ON_TIME);
  Serial.println(" ms)");
  delay(ON_TIME);
  
  // Pause OFF
  digitalWrite(LED_PIN, LOW);
  delay(OFF_TIME);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Connect LED **A** (Anode) to ESP32 **GPIO22**.
3. Connect LED **C** (Cathode) to ESP32 **GND**.
4. Paste the C++ code into the editor.
5. Select the default interpreted C++ runtime.
6. Click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Strobe Beacon Initialised.
Strobe! (200 ms)
Strobe! (200 ms)
Strobe! (200 ms)
```

## Expected Canvas Behavior
* The external LED flashes briefly (200 ms) and remains dark for 1 second.
* The terminal logs `Strobe! (200 ms)` on every pulse.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int LED_PIN = 22;` | Maps the external LED control to GPIO 22. |
| `delay(ON_TIME);` | Pauses execution for 200 ms while the LED is powered. |
| `delay(OFF_TIME);` | Pauses execution for 1000 ms while the LED is unpowered. |

## Hardware & Safety Concept: Current Limiting at 3.3V
The formula for calculating the current-limiting resistor at 3.3V logic level:
\[R = \frac{V_{supply} - V_{LED}}{I}\]
For a standard Red LED ($V_{LED} \approx 2.0V$) drawing $10\text{ mA}$:
\[R = \frac{3.3V - 2.0V}{0.010A} = 130\ \Omega\]
Using a **220 ohm** resistor is a safe, standard choice. It reduces the current draw to approximately 6 mA, keeping the ESP32 pin well within its 20 mA thermal limit while maintaining good visibility.

## Try This! (Challenges)
1. **Radar Pulse Pattern**: Alter the delay pattern to create a sweep effect: three rapid flashes followed by a 2-second delay.
2. **Reverse Duty Cycle**: Swap the variables (`ON_TIME = 1000` and `OFF_TIME = 200`). How does this affect the visual perception of the indicator?

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on and doesn't blink | Both delay calls missing | Verify the code has two distinct `delay()` statements: one after turning the pin HIGH, one after turning it LOW. |
| LED is too dim | Resistor value too high | Ensure you are using a 220 Ω or 330 Ω resistor. Using a 10k-ohm resistor will block almost all current, making the LED invisible. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 Onboard LED Blink](01-esp32-onboard-led-blink.md)
- [03 - ESP32 Alternate LED Blink](03-esp32-alternate-led-blink.md)
