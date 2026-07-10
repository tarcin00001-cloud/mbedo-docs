# 23 - ESP32 Slide Switch State Logging

Read the position of a two-position slide switch and log its state to the Serial Monitor in human-readable form.

## Goal
Learn how to read a slide (SPDT) switch using a digital input pin and print a descriptive label for each position rather than raw 0/1 values.

## What You Will Build
A slide switch wired to GPIO 4 with a 10 kΩ pull-down resistor. Sliding to one position drives the pin HIGH ("Mode A"); sliding to the other position lets the pull-down hold the pin LOW ("Mode B"). The state is printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Slide Switch (SPDT) | `switch` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Slide Switch | Common (C) | 3V3 | Red | Supply rail feeds the switch |
| Slide Switch | Normally Open (NO) | GPIO4 | Yellow | Signal pin — HIGH when switch in ON position |
| Slide Switch | Normally Closed (NC) | Not connected | — | Leave NC pin unconnected |
| 10 kΩ Resistor | Leg 1 | GPIO4 | White | Pull-down: holds pin LOW when switch in OFF position |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |

> **Wiring tip:** Connect the slide switch's Common (C) pin to 3V3 and the Normally Open (NO) pin to GPIO 4. When the switch is slid to the ON position it connects C to NO, pulling GPIO 4 HIGH. The 10 kΩ resistor holds GPIO 4 at LOW when the switch is in the OFF position. Leave the NC pin unconnected.

## Code
```cpp
// Slide Switch State Logger
const int SWITCH_PIN = 4;

void setup() {
  pinMode(SWITCH_PIN, INPUT);   // External pull-down fitted
  Serial.begin(115200);
  Serial.println("Slide Switch Logger ready.");
}

void loop() {
  int state = digitalRead(SWITCH_PIN);

  if (state == HIGH) {
    Serial.println("[SWITCH] Position: Mode A (ON)");
  } else {
    Serial.println("[SWITCH] Position: Mode B (OFF)");
  }

  delay(500);   // Log every 500 ms — readable rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Slide Switch** onto the canvas.
2. Connect Switch **signal output** to **GPIO4**.
3. Paste the code and click **Run**.
4. Toggle the slide switch widget on the canvas to change position.
5. Observe the Serial Monitor labels updating.

## Expected Output
Serial Monitor:
```
Slide Switch Logger ready.
[SWITCH] Position: Mode B (OFF)
[SWITCH] Position: Mode B (OFF)
[SWITCH] Position: Mode A (ON)
[SWITCH] Position: Mode A (ON)
```

## Expected Canvas Behavior
* The Serial Monitor logs the switch position every 500 ms.
* Toggling the slide switch widget immediately changes the label on the next print cycle.
* No LED is used — this project focuses on input reading and serial logging.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(SWITCH_PIN, INPUT)` | Configures GPIO 4 as a plain input — external pull-down keeps it LOW when the switch is open. |
| `int state = digitalRead(SWITCH_PIN)` | Reads the current position: HIGH when slide is in ON position. |
| `if (state == HIGH)` | Branches to print a human-readable label for the ON position. |
| `delay(500)` | Logs at 2 Hz — fast enough to see changes, slow enough to read comfortably. |

## Hardware & Safety Concept: Slide (SPDT) Switches
A Single-Pole Double-Throw (SPDT) slide switch has three pins: Common (C), Normally Closed (NC), and Normally Open (NO). In one slide position, C connects to NC; in the other, C connects to NO. This mechanical arrangement gives two stable positions with no bounce between them (unlike push buttons). Slide switches are used in electronics to select between two operating modes — for example, to switch a device between USB and battery power, or to enable/disable a safety lockout. Because they are mechanically stable, slide switches require no debounce logic in firmware.

## Try This! (Challenges)
1. **LED indicator**: Add an LED on GPIO 5 that lights when the switch is in Mode A.
2. **State change event**: Only print when the switch position changes, not every 500 ms.
3. **Mode-based behaviour**: In Mode A, blink an LED at 1 Hz; in Mode B, blink at 5 Hz.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Always reads HIGH | Pull-down missing or NC pin wired to GPIO 4 | Verify 10 kΩ is between GPIO 4 and GND; use the NO pin |
| Always reads LOW | Switch Common not connected to 3V3 | Move the Common wire to 3V3 |
| Reading changes randomly | Floating NC pin interfering | Ensure NC pin is left unconnected (or tie to GND) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 Button-Controlled LED](16-esp32-button-controlled-led.md)
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
