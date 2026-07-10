# 29 - Pico Vibration Sensor

Detect physical impacts and latch an alarm using a vibration sensor.

## Goal
Learn how to read high-speed shock/vibration inputs and latch an alarm state in software until a manual reset is triggered.

## What You Will Build
A security vibration alarm:
- **Vibration Sensor (GP16)**: Monitored via digital input.
- **Push Button (GP17)**: Configured with internal pull-up as a Reset button.
- **Red LED (GP15)**: Latches ON (Red alert) if any vibration is detected.
- **Green LED (GP14)**: Glows when the system is armed and quiet.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| SW-18010P Vibration Sensor | `button` | Yes (represented by switch button) | Yes |
| Push Button | `button` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Optional | Yes (two resistors) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Vibration Sensor | Terminal 1 | GP16 | Sense pin |
| Vibration Sensor | Terminal 2 | GND | Ground return |
| Reset Button | Terminal 1 | GP17 | Reset trigger |
| Reset Button | Terminal 2 | GND | Ground return |
| Green LED | Anode | GP14 | Armed status |
| Green LED | Cathode | GND | Ground return via resistor |
| Red LED | Anode | GP15 | Alarm status |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int VIB_PIN    = 16;
const int RESET_PIN  = 17;
const int ARMED_LED  = 14;
const int ALARM_LED  = 15;

bool alarmActive = false;

void setup() {
  pinMode(VIB_PIN, INPUT_PULLUP);
  pinMode(RESET_PIN, INPUT_PULLUP);
  pinMode(ARMED_LED, OUTPUT);
  pinMode(ALARM_LED, OUTPUT);

  // Initialize state
  digitalWrite(ARMED_LED, HIGH);
  digitalWrite(ALARM_LED, LOW);
}

void loop() {
  int vibState = digitalRead(VIB_PIN);
  int resetState = digitalRead(RESET_PIN);

  // Shock detected: vibration switch closed briefly (LOW)
  if (vibState == LOW) {
    alarmActive = true; // Latch alarm state
  }

  // Reset button pressed (LOW)
  if (resetState == LOW) {
    alarmActive = false; // Clear alarm state
  }

  // Update output indicators
  if (alarmActive) {
    digitalWrite(ARMED_LED, LOW);
    digitalWrite(ALARM_LED, HIGH); // Red ON
  } else {
    digitalWrite(ARMED_LED, HIGH); // Green ON
    digitalWrite(ALARM_LED, LOW);
  }

  delay(20);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons** (one representing the vibration sensor, one the Reset button), **Green LED**, and **Red LED** onto the canvas.
2. Connect Vibration Switch: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect Reset Button: **Terminal 1** to **GP17**, **Terminal 2** to **GND**.
4. Connect Green LED Anode to **GP14**, Red LED Anode to **GP15**, Cathodes to **GND**.
5. Paste code, select the interpreted mode, and click **Run**.
6. Click the Vibration Switch (GP16) once to trigger the Red alarm, and click Reset (GP17) to clear it.

## Expected Output

Terminal:
```
Simulation active. Vibration security system online.
```

## Expected Canvas Behavior
| Event | Green LED (GP14) | Red LED (GP15) | System State |
| --- | --- | --- | --- |
| Startup | HIGH (ON) | LOW (OFF) | Armed & Quiet |
| Trigger (GP16 LOW) | LOW (OFF) | HIGH (ON) | **Alarm Latched** |
| Release GP16 | LOW (OFF) | HIGH (ON) | **Alarm Latched** |
| Reset (GP17 LOW) | HIGH (ON) | LOW (OFF) | Armed & Quiet |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `alarmActive = true` | Latches the alarm boolean flag to true, locking the alert state even after the vibration contact opens again. |

## Hardware & Safety Concept: SW-18010P Sensor
The SW-18010P vibration sensor contains a central metal rod surrounded by a loose spring. Any physical impact or vibration causes the spring to move and contact the central rod, closing the circuit briefly. Because these contact closures happen in microseconds, the microcontroller must poll the pin rapidly to avoid missing the shock event.

## Try This! (Challenges)
1. **Siren Warning**: Sound a buzzer on GP10 in a rapid pulsing sequence when the alarm is latched.
2. **Auto-Reset**: Modify the code to clear the alarm automatically after 5 seconds if no further vibration is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers randomly | Sensitivity too high | Mechanical vibration switches are highly sensitive. Ensure the Pico board is placed on a completely flat, stable surface. |

## Mode Notes
This basic state-latching project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [28 - Pico Tilt Switch](28-pico-tilt-switch.md)
