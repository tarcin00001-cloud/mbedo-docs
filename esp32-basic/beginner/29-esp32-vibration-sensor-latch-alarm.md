# 29 - ESP32 Vibration Sensor Latch Alarm

Use a vibration sensor (SW-420 module) to detect impact or vibration and trigger a latching alarm that stays active until manually reset.

## Goal
Learn how to implement a **software latch** that captures a brief event and holds an alarm output on even after the trigger disappears — a critical pattern for security and monitoring systems.

## What You Will Build
A SW-420 vibration sensor on GPIO 4. Any detected vibration sets a software latch. A red LED on GPIO 5 and an active buzzer on GPIO 15 stay on until a reset button on GPIO 16 is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| SW-420 Vibration Sensor Module | `vibration_sensor` | Yes | Yes |
| Tactile Push Button (reset) | `button` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SW-420 Module | VCC | 3V3 | Red | Module supply voltage |
| SW-420 Module | GND | GND | Black | Module ground |
| SW-420 Module | DO (digital out) | GPIO4 | Yellow | HIGH on vibration detected |
| Reset Button | Pin 1 (supply side) | 3V3 | Red | Supply for reset button |
| Reset Button | Pin 2 (signal side) | GPIO16 | Blue | Reset input |
| 10 kΩ Resistor | Leg 1 | GPIO16 | White | Pull-down for reset button |
| 10 kΩ Resistor | Leg 2 | GND | Black | Ground reference |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Latch alarm indicator |
| LED (red) | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm buzzer |
| Active Buzzer | GND (−) | GND | Black | Ground return |

> **Wiring tip:** The SW-420 module has a sensitivity trimmer potentiometer. Turn it clockwise for higher sensitivity (triggers on light vibration) and counter-clockwise for lower sensitivity (only triggers on strong impacts). The module's DO output goes HIGH on vibration. GPIO 16 (ESP32) is available as a general-purpose I/O and works well as a reset input.

## Code
```cpp
// Vibration Sensor Latch Alarm
const int VIB_PIN    = 4;    // SW-420 digital output
const int RESET_PIN  = 16;   // Reset button
const int LED_PIN    = 5;    // Alarm LED
const int BUZZER_PIN = 15;   // Alarm buzzer

bool alarmLatched = false;   // Software latch state

void setup() {
  pinMode(VIB_PIN,    INPUT);         // SW-420 drives this pin
  pinMode(RESET_PIN,  INPUT);         // External pull-down on reset button
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Vibration Latch Alarm ready — monitoring...");
}

void loop() {
  // Set the latch if vibration is detected
  if (digitalRead(VIB_PIN) == HIGH) {
    if (!alarmLatched) {
      alarmLatched = true;
      Serial.println("!! VIBRATION DETECTED — ALARM LATCHED !!");
    }
  }

  // Reset the latch when the reset button is pressed
  if (digitalRead(RESET_PIN) == HIGH) {
    if (alarmLatched) {
      alarmLatched = false;
      Serial.println("Alarm reset by operator.");
    }
  }

  // Drive outputs based on latch state
  digitalWrite(LED_PIN,    alarmLatched ? HIGH : LOW);
  digitalWrite(BUZZER_PIN, alarmLatched ? HIGH : LOW);

  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Vibration Sensor**, **Button** (reset), **LED**, and **Buzzer** onto the canvas.
2. Connect Vibration Sensor **DO** to **GPIO4**, Reset Button to **GPIO16**.
3. Connect LED **anode** to **GPIO5**, Buzzer **VCC** to **GPIO15**.
4. Paste the code and click **Run**.
5. Click the vibration sensor widget to trigger the alarm.
6. Click the reset button widget to clear the latch.

## Expected Output
Serial Monitor:
```
Vibration Latch Alarm ready — monitoring...
!! VIBRATION DETECTED — ALARM LATCHED !!
!! VIBRATION DETECTED — ALARM LATCHED !!
Alarm reset by operator.
!! VIBRATION DETECTED — ALARM LATCHED !!
Alarm reset by operator.
```

## Expected Canvas Behavior
* LED and buzzer are off at startup.
* Clicking the vibration sensor widget lights the LED and activates the buzzer permanently.
* Clicking the reset button clears both outputs.
* A second vibration re-latches the alarm immediately.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `bool alarmLatched = false` | Software latch variable — holds the alarm state independently of the sensor. |
| `if (digitalRead(VIB_PIN) == HIGH)` | Detects an active vibration reading (HIGH) from the SW-420 module. |
| `alarmLatched = true` | Sets the latch — alarm stays active even after the vibration pulse ends. |
| `if (digitalRead(RESET_PIN) == HIGH)` | Checks for manual reset button press — clears the latch. |
| `alarmLatched ? HIGH : LOW` | Ternary operator: drives outputs HIGH when latched, LOW when cleared. |

## Hardware & Safety Concept: Latching Alarms in Security Systems
Real-world security alarms must **hold the alarm state** after the triggering event — because an intruder triggering a motion sensor and immediately moving away should not cause the alarm to silently stop. A software latch achieves this: a single HIGH pulse from the sensor sets a boolean that keeps the alarm outputs energised until a deliberate human reset action clears it. This pattern is used in: building intruder alarms, fire alarm panels, industrial machine fault indicators, and vehicle anti-theft systems. The reset button represents the **authorised operator** action required to confirm the alarm has been acknowledged and investigated.

## Try This! (Challenges)
1. **Alarm log count**: Count how many times the alarm has been latched during a session and print it on each reset.
2. **Auto-reset timer**: If the latch is not manually reset within 30 seconds, auto-reset it to avoid lockout.
3. **Sensitivity print**: Read the sensor output 100 times and print the percentage of time it was HIGH to estimate ambient vibration level.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm latches immediately at startup | SW-420 sensitivity too high | Turn the trimmer potentiometer counter-clockwise to reduce sensitivity |
| Alarm never triggers | Module DO wire loose | Reseat the DO-to-GPIO4 wire; check module VCC and GND |
| Reset button does not work | Pull-down missing on GPIO 16 | Add 10 kΩ between GPIO 16 and GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [28 - ESP32 Tilt Sensor Level Indicator](28-esp32-tilt-sensor-level-indicator.md)
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
- [25 - ESP32 Magnetic Switch Gate Alert](25-esp32-magnetic-switch-gate-alert.md)
