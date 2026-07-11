# 106 - Shake Alarm System

Build a secure intrusion or seismic alarm that detects physical shaking using a vibration sensor, latches the alarm state, and requires a manual button press to reset the active buzzer.

## Goal
Learn how to interface digital vibration/shock sensors, implement software-based state latching, and clear alarm states using a pull-up reset button.

## What You Will Build
A tamper-proof shake detector. When the device experiences a physical vibration or shock, the sensor triggers a latch state. Once latched, the active buzzer sounds continuously, ignoring further sensor inputs until the user presses the reset button.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Vibration / SW-420 Sensor | `vibration_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Push Button (Reset) | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Vibration Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Vibration Sensor | GND | GND | Black | Ground reference |
| Vibration Sensor | DO (Digital Out)| GPIO 17 | Green | Active HIGH shock signal |
| Active Buzzer | VCC | GPIO 14 | Orange | Output high triggers alarm |
| Active Buzzer | GND | GND | Black | Ground reference |
| Push Button | Pin 1 | GPIO 16 | Blue | Input pin with internal pull-up |
| Push Button | Pin 2 | GND | Black | Ground connection |

> **Wiring tip:** The SW-420 vibration sensor output goes `HIGH` when a physical vibration occurs. The reset push button is configured with an internal pull-up resistor, meaning it reads `LOW` when pressed.

## Code
```cpp
const int VIB_PIN = 17;      // Vibration sensor on GPIO 17
const int BUZZER_PIN = 14;   // Active Buzzer on GPIO 14
const int BUTTON_PIN = 16;   // Reset Button on GPIO 16

int isAlarmLatched = 0;      // Latch state variable (0 = Safe, 1 = Alarm)

void setup() {
  pinMode(VIB_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP); // Use internal pull-up resistor

  digitalWrite(BUZZER_PIN, LOW); // Start with buzzer OFF
}

void loop() {
  int vibration = digitalRead(VIB_PIN);
  int buttonState = digitalRead(BUTTON_PIN);

  // If vibration is detected, latch the alarm
  if (vibration == HIGH) {
    isAlarmLatched = 1;
  }

  // If reset button is pressed (LOW), clear the alarm latch
  if (buttonState == LOW) {
    isAlarmLatched = 0;
  }

  // Drive the buzzer according to the latch status
  if (isAlarmLatched == 1) {
    digitalWrite(BUZZER_PIN, HIGH); // Alarm active
  } else {
    digitalWrite(BUZZER_PIN, LOW);  // Alarm cleared
  }

  delay(50); // High polling rate for shock detection
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Vibration Sensor**, **Active Buzzer**, and **Push Button** components onto the canvas.
2. Wire the Vibration Sensor: **VCC** to **3V3**, **GND** to **GND**, and **DO** to **GPIO 17**.
3. Wire the Active Buzzer: **VCC** to **GPIO 14** and **GND** to **GND**.
4. Wire the Push Button: **Pin 1** to **GPIO 16** and **Pin 2** to **GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Tamper Status: SECURE
...
Tamper Status: SHOCK DETECTED - ALARM LATCHED!
...
Tamper Status: RESET COMMAND RECEIVED - CLEARING...
```

## Expected Canvas Behavior
* Tapping or triggering the simulated vibration sensor changes the sensor output state to high.
* The active buzzer turns ON immediately and stays ON, even if the vibration sensor is no longer active.
* Pressing the push button turns the active buzzer OFF.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures GP16 as an input and enables the internal 50k-ohm pull-up resistor to pull it to 3.3V when idle. |
| `digitalRead(VIB_PIN)` | Samples the vibration sensor module output. |
| `isAlarmLatched = 1` | Sets the persistent flag to store the alert status. |
| `buttonState == LOW` | Identifies when the button connects to ground (pressed). |

## Hardware & Safety Concept
* **Spring-mass Sensor Principle**: Mechanical vibration sensors (like the SW-420) contain a tiny internal spring wrapped around a metal pin. When still, the spring touches the pin, completing the circuit. Under vibration, the spring flexes away, breaking the contact momentarily. An onboard comparator IC (LM393) converts these brief breaks into clean digital logic pulses.
* **Latch Mechanism Logic**: In security contexts, sensor signals are often transient (lasting milliseconds). If the alarm only sounded while vibration was occurring, an intruder could move the device and silence it. Latching ensures the security response is persistent until authorized reset occurs.

## Try This! (Challenges)
1. **Warning Light**: Flash the onboard Red LED (`LED_R` on GPIO 23) at 200 ms intervals while the alarm is latched to provide a visual warning.
2. **Double Button Reset**: Require the user to press the reset button twice within 1 second to clear the alarm, preventing accidental cancellation.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers randomly without vibration | Sensitivity threshold is set too low | Adjust the physical potentiometer on the SW-420 sensor to decrease sensitivity. |
| Reset button does not turn off the alarm | Pin mode not set to INPUT_PULLUP | Ensure `pinMode(BUTTON_PIN, INPUT_PULLUP)` is declared so the pin does not float. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [70 - Active Buzzer Chime Scale Generator](70-active-buzzer-chime-scale-generator.md)
- [98 - Gas Leakage Alarm Node](98-gas-leakage-alarm-node.md)
