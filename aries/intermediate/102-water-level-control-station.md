# 102 - Water Level Control Station

Build an automated water level regulator that monitors depth using an analog sensor, control a filling pump via a relay, and sound an overflow alarm using a buzzer on the VEGA ARIES v3 board.

## Goal
Learn how to read resistive analog depth sensors, construct multi-state control systems with thresholds, and operate relays and buzzer alarms safely.

## What You Will Build
A reservoir management system. When the water level drops below a low threshold, the pump relay activates to refill the tank. If the water level rises above a high threshold, the relay shuts off immediately to prevent flooding, and a buzzer sounds an active alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Water Level Sensor | `water_sensor` | Yes | Yes |
| 5V Relay Module (Pump) | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Water Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Water Sensor | GND | GND | Black | Ground reference |
| Water Sensor | OUT (Signal) | ADC2 (GP28) | White | Analog depth level output |
| Relay Module | VCC | 5V | Red | Relay operating voltage |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 15 | Blue | Control signal pin |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers alarm |
| Active Buzzer | GND | GND | Black | Ground reference |

> **Wiring tip:** Resistive water sensors work by measuring electricity conducted across parallel exposed trace lines. Power the water sensor from 3.3V and ensure proper ground reference to maintain clean readings.

## Code
```cpp
const int WATER_PIN = 28;  // ADC2 is GP28
const int RELAY_PIN = 15;  // Pump relay on GPIO 15
const int BUZZER_PIN = 14; // Alarm buzzer on GPIO 14

const int LOW_LEVEL = 1000;  // Level below which pump turns ON
const int HIGH_LEVEL = 3000; // Level above which overflow alarm sounds

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Initialize both off
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  int level = analogRead(WATER_PIN);

  if (level < LOW_LEVEL) {
    // Water level too low: start pumping water
    digitalWrite(RELAY_PIN, HIGH);
    digitalWrite(BUZZER_PIN, LOW);
  } 
  else if (level > HIGH_LEVEL) {
    // Water level overflowing: stop pump immediately and alarm
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, HIGH);
  } 
  else {
    // Normal water level: stop alarm, relay stays in current state
    digitalWrite(BUZZER_PIN, LOW);
  }

  delay(250); // Level reading interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Water Level Sensor**, **5V Relay Module**, and **Active Buzzer** components onto the canvas.
2. Wire the Water Sensor: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **ADC2 (GP28)**.
3. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
4. Wire the Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Water Level: 850 - PUMP ACTIVE (REFILLING)
...
Water Level: 2100 - NORMAL
...
Water Level: 3200 - OVERFLOW DETECTED - ALARM ACTIVE!
```

## Expected Canvas Behavior
* Sliding the simulated water level slider down below 1000 turns on the relay indicator.
* Sliding the level up above 3000 turns off the relay and makes the active buzzer play a high-pitched alarm sound.
* Positioning the slider between 1000 and 3000 turns off the buzzer and maintains the current system safety level.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(WATER_PIN)` | Samples the resistive voltage of the water sensor probes (0 to 4095). |
| `level < LOW_LEVEL` | Detects dry conditions and turns on the relay output pin GP15. |
| `level > HIGH_LEVEL` | Detects overflow conditions to turn off the pump and trigger the alarm. |
| `digitalWrite(BUZZER_PIN, LOW)` | Disables the buzzer alarm when water level settles in the safe zone. |

## Hardware & Safety Concept
* **Liquid Corrosion & DC Bias**: Passing direct current (DC) through metal probes in contact with water causes electrochemical corrosion (electrolysis), which degrades the probes in a few weeks. In practice, power the sensor only during measurements or use alternating current (AC) excitation.
* **Fail-Safe Design**: When designing physical water pump systems, use normally closed (NC) or normally open (NO) contacts appropriately so that if the microcontroller loses power, the relay defaults to a position that turns the water pump OFF to prevent floods.

## Try This! (Challenges)
1. **Low Level Alarm**: If the water level drops extremely low (e.g., below 300), sound a different warning (e.g. beep the buzzer intermittently) to indicate that the source reservoir is empty.
2. **Pump Timeout**: Implement a safety counter that automatically shuts off the relay if the pump runs continuously for more than 10 seconds without the water level rising, preventing motor burnout.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor readings fluctuate wildly | Electrical noise in water or loose ground | Ensure common ground is wired tightly, and place a 0.1uF capacitor across sensor VCC and GND. |
| Relay does not toggle | Relay requires higher drive voltage | Ensure the relay VCC is connected to the 5V output of the ARIES board. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [98 - Gas Leakage Alarm Node](98-gas-leakage-alarm-node.md)
- [110 - Soil Moisture Automated Irrigation](110-soil-moisture-automated-irrigation.md)
