# 12 - ESP32 Relay Safety Signal Warning

Build an industrial safety controller that sounds an audible and visual warning strobe before toggling a high-power relay switch.

## Goal
Learn how to create compound sequencing routines, integrate warning signals (LED + Buzzer) with relay control outputs, and implement industrial safety concepts.

## What You Will Build
A safety contactor system: before a Relay (GPIO 15) closes, a Warning LED (GPIO 22) and Buzzer (GPIO 25) pulse three times rapidly (safety pre-warning). The relay then holds closed for 4 seconds. Before opening, the warnings flash three times again.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Optional | Yes (LED current limit) |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO15 | Orange | Relay control input pin |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Red LED | Anode (via 220 Ω) | GPIO22 | Red | Warning LED output pin |
| Red LED | Cathode | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO25 | Blue | Warning audio output pin |
| Active Buzzer | GND (-) | GND | Black | Ground return |
| All Modules | GND | GND | Black | Shared return rail to ground |

> **Wiring tip:** The LED, Buzzer, and Relay must all connect to GND. Ensure the Relay VCC is connected to the 5V (VBUS) pin.

## Code
```cpp
// Hardware pin mappings
const int RELAY_PIN = 15;
const int WARNING_LED = 22;
const int WARNING_BUZZER = 25;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  pinMode(WARNING_BUZZER, OUTPUT);
  
  // Set default safe states (everything OFF)
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(WARNING_LED, LOW);
  digitalWrite(WARNING_BUZZER, LOW);
  
  Serial.begin(115200);
  Serial.println("ESP32 Safety Switch Monitor Online.");
}

// Function to flash warnings (3 quick pulses)
void soundSafetyWarning() {
  Serial.println(">> WARNING: Toggling relay state in 1 second! <<");
  for (int i = 0; i < 3; i++) {
    digitalWrite(WARNING_LED, HIGH);
    digitalWrite(WARNING_BUZZER, HIGH);
    delay(100);
    digitalWrite(WARNING_LED, LOW);
    digitalWrite(WARNING_BUZZER, LOW);
    delay(100);
  }
  delay(500); // Small pause before relay action
}

void loop() {
  // 1. Alert that relay is closing, then close it
  soundSafetyWarning();
  digitalWrite(RELAY_PIN, HIGH);
  Serial.println("Relay State: CLOSED (ON)");
  delay(4000); // Keep on for 4 seconds
  
  // 2. Alert that relay is opening, then open it
  soundSafetyWarning();
  digitalWrite(RELAY_PIN, LOW);
  Serial.println("Relay State: OPEN (OFF)");
  delay(4000); // Keep off for 4 seconds
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, **LED**, and **Buzzer** onto the canvas.
2. Connect Relay to **GPIO15**, LED to **GPIO22**, and Buzzer to **GPIO25**.
3. Connect VCC/GND lines as shown in the wiring table.
4. Paste the C++ code into the editor.
5. Select interpreted C++ mode and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Safety Switch Monitor Online.
>> WARNING: Toggling relay state in 1 second! <<
Relay State: CLOSED (ON)
>> WARNING: Toggling relay state in 1 second! <<
Relay State: OPEN (OFF)
```

## Expected Canvas Behavior
* Startup: Red LED and Buzzer flash/beep three times quickly.
* The Relay switches to CLOSED (switch arm clicks closed).
* 4 seconds: Red LED and Buzzer flash/beep three times again.
* The Relay switches to OPEN.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `void soundSafetyWarning()` | Defines a helper loop that cycles the warning outputs three times. |
| `digitalWrite(RELAY_PIN, HIGH);` | Closes the relay contacts after the safety alert finishes. |

## Hardware & Safety Concept: Pre-Start Warning Systems
In industrial automation (such as conveyor belts, robotic arms, or heavy machinery), starting a motor without warning is a major safety hazard. If operators are working near the machine, they can be injured. Standard safety regulations require a **pre-start delay** combined with a flashing strobe and horn to give personnel time to clear the area before high-power actuators energize.

## Try This! (Challenges)
1. **Pulsing Ticker**: During the 4-second active state, flash the warning LED slowly (once per second) to indicate that the circuit is currently energized.
2. **Frequency Shift**: Add a second button on GPIO 4. If clicked, double the speed of the pre-start alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound but LED flashes | Pin mismatch or bad ground | Verify the buzzer VCC is wired to GPIO 25. Check that the buzzer has a solid connection to the GND rail. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [11 - ESP32 Relay Module Control](11-esp32-relay-module-control.md)
- [13 - ESP32 5V DC Fan Toggle](13-esp32-5v-dc-fan-toggle.md)
