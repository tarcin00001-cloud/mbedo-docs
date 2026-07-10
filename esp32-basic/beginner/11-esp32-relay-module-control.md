# 11 - ESP32 Relay Module Control

Build an automated power switcher that drives a relay module to open and close a circuit.

## Goal
Learn how to interface an electromagnetic relay module with the ESP32, understand how relays actuate high-power loads, and write digital control signals in C++.

## What You Will Build
A relay module connected to GPIO 15 toggles states (ON/Closed for 2 seconds, OFF/Open for 2 seconds) repeatedly, simulating an automated main switch.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes (5V relay module with optocoupler) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO15 | Orange | Relay control input pin |
| Relay Module | VCC | 5V (VBUS) | Red | Power line (5V required for coil) |
| Relay Module | GND | GND | Black | Ground return path |

> **Wiring tip:** A 5V relay module requires 5V to actuate its internal electromagnet. Connect its VCC pin to the ESP32 VBUS/5V pin. Connect GND to GND and the IN signal pin to GPIO 15.

## Code
```cpp
// Relay signal pin connected to GPIO 15
const int RELAY_PIN = 15;

const int SWITCH_INTERVAL = 2000; // Switch state every 2 seconds

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  
  Serial.begin(115200);
  Serial.println("ESP32 Relay Controller Online.");
}

void loop() {
  // Turn relay ON (Close contacts)
  digitalWrite(RELAY_PIN, HIGH);
  Serial.println("Relay State: CLOSED (ON) -> Current Flowing");
  delay(SWITCH_INTERVAL);
  
  // Turn relay OFF (Open contacts)
  digitalWrite(RELAY_PIN, LOW);
  Serial.println("Relay State: OPEN (OFF) -> Circuit Broken");
  delay(SWITCH_INTERVAL);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Relay** onto the canvas.
2. Connect Relay **IN** to **GPIO15**.
3. Connect Relay **VCC** to ESP32 **5V** (VBUS).
4. Connect Relay **GND** to ESP32 **GND**.
5. Paste code, select interpreted C++ mode, and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Relay Controller Online.
Relay State: CLOSED (ON) -> Current Flowing
Relay State: OPEN (OFF) -> Circuit Broken
```

## Expected Canvas Behavior
* The relay switches state: the switch arm moves visually on the canvas, and a "click" log is generated.
* The state changes are logged to the Serial Monitor.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int RELAY_PIN = 15;` | Identifies GPIO 15 as the relay driver signal pin. |
| `digitalWrite(RELAY_PIN, HIGH);` | Drives the IN pin of the relay module HIGH, triggering the internal transistor to energize the coil. |

## Hardware & Safety Concept: Opto-isolation and High Voltage
Electromagnetic relays contain a coil that draws around **70–100 mA** when energized, which exceeds the ESP32 pin limit of 20 mA. Therefore, **relay modules** include a driving transistor. Many also feature an **optocoupler**, which uses an internal infrared LED and light detector to isolate the ESP32 electrically from the high-voltage load. This prevents inductive voltage spikes from entering and burning out the microcontroller.

## Try This! (Challenges)
1. **Indicator sync**: Add an external Red LED on GPIO 22 and configure it to turn ON when the relay is closed (active power indicator).
2. **Cycle Counter**: Count the number of switching operations and print it to the Serial Monitor, alerting when the relay has clicked 50 times.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay LED turns on but contacts do not click | Coil power insufficient | Check that the relay VCC is connected to the ESP32 **5V (VBUS)** pin. Running a 5V relay from a 3.3V pin will not provide enough magnetic force to move the mechanical contact arm. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - ESP32 Relay Safety Signal Warning](12-esp32-relay-safety-signal-warning.md)
- [13 - ESP32 5V DC Fan Toggle](13-esp32-5v-dc-fan-toggle.md)
