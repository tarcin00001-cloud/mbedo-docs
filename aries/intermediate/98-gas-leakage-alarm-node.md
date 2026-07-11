# 98 - Gas Leakage Alarm Node

Build a safety monitor that detects flammable gas leakage using an MQ-2 gas sensor, triggering an audible alarm and a relay to start ventilation.

## Goal
Learn how to interface an analog gas sensor with the ARIES board, execute threshold-based alarm logic, and control high-power loads (like ventilation fans) via a relay module without loops.

## What You Will Build
An automatic safety system that reads gas concentrations. When levels exceed a set threshold, a buzzer sounds to alert occupants, and a relay activates (simulating turning on an exhaust fan) until gas levels drop back to safety.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 5V Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Red | MQ-2 heater/circuit power (5V) |
| MQ-2 Sensor | GND | GND | Black | Ground reference |
| MQ-2 Sensor | AO | ADC2 (GP28) | White | Analog gas level output |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers alarm |
| Active Buzzer | GND | GND | Black | Ground reference |
| Relay Module | VCC | 5V | Red | Relay coil operating voltage |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 15 | Blue | Control input (Active High) |

> **Wiring tip:** The MQ-2 internal heater requires 5V to function correctly. Make sure VCC is connected to the ARIES 5V pin. The active buzzer can be triggered directly using GPIO 14.

## Code
```cpp
const int MQ2_PIN = 28;      // ADC2 is GP28
const int BUZZER_PIN = 14;   // Active Buzzer on GPIO 14
const int RELAY_PIN = 15;    // Safety Relay on GPIO 15

// Threshold value for gas detection (out of 4095)
const int GAS_THRESHOLD = 1500; 

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  // Start with alarm and relay off
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
}

void loop() {
  int gasLevel = analogRead(MQ2_PIN);

  if (gasLevel > GAS_THRESHOLD) {
    digitalWrite(BUZZER_PIN, HIGH); // Alarm ON
    digitalWrite(RELAY_PIN, HIGH);  // Exhaust Fan ON
  } else {
    digitalWrite(BUZZER_PIN, LOW);  // Alarm OFF
    digitalWrite(RELAY_PIN, LOW);   // Exhaust Fan OFF
  }

  delay(200); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MQ-2 Gas Sensor**, **Active Buzzer**, and **5V Relay Module** components onto the canvas.
2. Wire the MQ-2 Sensor: **VCC** to **5V**, **GND** to **GND**, and **AO** to **ADC2 (GP28)**.
3. Wire the Active Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
4. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
MQ-2 Raw Analog: 850 (Normal)
...
MQ-2 Raw Analog: 1680 (GAS LEAK DETECTED - ALARM ACTIVE!)
```

## Expected Canvas Behavior
* Increasing the gas concentration slider on the simulated MQ-2 sensor triggers the active buzzer to play a continuous tone.
* The relay switches states (indicated by a click or visual change on the canvas), representing the activation of the emergency exhaust system.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Configures GPIO 14 to drive the active buzzer. |
| `analogRead(MQ2_PIN)` | Reads the raw gas sensor concentration value (0 to 4095). |
| `gasLevel > GAS_THRESHOLD` | Evaluates if the current reading exceeds the safety limit (1500). |
| `digitalWrite(BUZZER_PIN, HIGH)` | Applies 3.3V to the buzzer's control pin to make it sound. |
| `digitalWrite(RELAY_PIN, LOW)` | Removes power from the relay coil to shut off ventilation once levels return to normal. |

## Hardware & Safety Concept
* **MQ-2 Heating Phase**: The MQ-2 gas sensor uses an internal heating coil to warm up its tin dioxide (SnO2) sensing element. Upon cold start on physical hardware, the sensor draws significant current (~150mA) and requires a preheating period of 24–48 hours before outputting completely stable calibration readings.
* **Galvanic Isolation**: A relay provides galvanic isolation between the low-voltage ARIES control circuit (3.3V/5V) and the high-voltage load circuit (e.g. 230V mains fan). This prevents feedback noise and electrical surges from reaching the microcontroller.

## Try This! (Challenges)
1. **Flashing Visual Warning**: Blink the onboard red LED (`LED_R` on GPIO 23) rapidly at 100 ms intervals when the gas alarm is triggered.
2. **Hysteresis Implementation**: To prevent the relay from constantly clicking when the gas level floats exactly around 1500, implement a threshold gap (turn on at 1500, turn off only when gas drops below 1300).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds faint or clicks instead of buzzing | Powered directly from low-current digital pin | Ensure the buzzer is an active buzzer, or use a transistor driver circuit if using a passive buzzer. |
| Relay does not engage even when threshold is met | Input current too low or ground not shared | Verify the relay module's VCC is connected to ARIES 5V, not 3V3, and that grounds are connected. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [70 - Active Buzzer Chime Scale Generator](70-active-buzzer-chime-scale-generator.md)
- [102 - Water Level Control Station](102-water-level-control-station.md)
