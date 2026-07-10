# 44 - ESP32 Flame Sensor Digital Fire Alert

Detect the presence of an open flame using a digital flame sensor module and immediately trigger a multi-output fire alert.

## Goal
Learn how a flame sensor detects infrared light in the 700–1000 nm wavelength range emitted by fire, how to read its active-LOW digital output, and how to drive an emergency alert system.

## What You Will Build
A flame sensor module on GPIO 4 (using `INPUT_PULLUP`). When a flame is detected, DO goes LOW. The ESP32 immediately lights a red LED on GPIO 5 and sounds a buzzer on GPIO 15 with a rapid alarm pattern.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Flame Sensor Module (IR type) | `flame_sensor` | Yes | Yes |
| LED (red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor Module | VCC | 3V3 | Red | Module supply voltage |
| Flame Sensor Module | GND | GND | Black | Module ground |
| Flame Sensor Module | DO (digital out) | GPIO4 | Yellow | Active-LOW — LOW when flame detected |
| LED (red) | Anode (+) | GPIO5 via 330 Ω | Orange | Fire alert LED |
| LED (red) | Cathode (−) | GND | Black | Ground return |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Fire alarm buzzer |
| Active Buzzer | GND (−) | GND | Black | Ground return |

> **Wiring tip:** Flame sensor modules contain a phototransistor sensitive to infrared radiation (700–1000 nm). The module uses active-LOW logic on DO: LOW = flame present (IR threshold exceeded), HIGH = no flame. Keep the sensor away from incandescent lamps and direct sunlight, which also emit significant IR. The sensitivity trimmer adjusts detection range (1–100 cm depending on flame size).

## Code
```cpp
// Flame Sensor Digital Fire Alert — active-LOW output
const int FLAME_PIN  = 4;
const int LED_PIN    = 5;
const int BUZZER_PIN = 15;

void setup() {
  pinMode(FLAME_PIN,  INPUT_PULLUP); // Active-LOW — LOW = flame
  pinMode(LED_PIN,    OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN,    LOW);
  digitalWrite(BUZZER_PIN, LOW);

  Serial.begin(115200);
  Serial.println("Flame Sensor Alert ready — monitoring for fire.");
}

void loop() {
  int flameState = digitalRead(FLAME_PIN);

  if (flameState == LOW) {
    // Flame detected — active-LOW output pulled to GND
    Serial.println("!! FIRE DETECTED — ALARM !!");

    // Rapid triple beep alarm
    for (int i = 0; i < 3; i++) {
      digitalWrite(LED_PIN,    HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(LED_PIN,    LOW);
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    }
  } else {
    Serial.println("No flame — safe.");
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Flame Sensor**, **LED**, and **Buzzer** onto the canvas.
2. Connect Flame Sensor **DO** to **GPIO4**, LED **anode** to **GPIO5**, Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Toggle the flame sensor widget to simulate fire detection.

## Expected Output
Serial Monitor:
```
Flame Sensor Alert ready — monitoring for fire.
No flame — safe.
No flame — safe.
!! FIRE DETECTED — ALARM !!
!! FIRE DETECTED — ALARM !!
No flame — safe.
```

## Expected Canvas Behavior
* LED and buzzer are idle when the flame sensor widget is inactive.
* Activating the flame sensor widget triggers rapid three-flash LED bursts and buzzer pulses.
* Returning to no-flame state stops the alarm after the current triple-beep cycle completes.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `pinMode(FLAME_PIN, INPUT_PULLUP)` | Internal pull-up holds pin HIGH; flame detection pulls it to GND (LOW). |
| `if (flameState == LOW)` | Active-LOW detection: fires the alarm when DO is pulled LOW by the module. |
| `for (int i = 0; i < 3; i++)` | Generates three alarm flashes/beeps in rapid succession per loop iteration. |
| `delay(150) / delay(100)` | ON time (150 ms) and OFF gap (100 ms) between each beep in the alarm burst. |

## Hardware & Safety Concept: Infrared Flame Detection
Open flames emit a characteristic infrared spectral peak around 760 nm (near-infrared). The flame sensor module uses a **photodiode** or **phototransistor** filtered to pass only this wavelength range. When flame IR reaches the sensor, the phototransistor conducts and the module's comparator asserts DO LOW. Detection range depends on flame size and ambient IR levels — a candle flame can be detected at ~20 cm; a gas flame at ~80 cm. In real fire alarm systems, flame detectors are used alongside heat detectors and smoke detectors in a multi-sensor "AND" configuration — a fire is declared only when multiple sensor types agree. This reduces false alarms from heaters, sunlight, and welding operations.

## Try This! (Challenges)
1. **Latching alarm**: Once fire is detected, keep the alarm active until a separate reset button is pressed.
2. **Relay cutoff**: Add a relay on GPIO 16 wired to cut power to a gas valve when fire is detected.
3. **Distance estimation**: Connect the AO pin to GPIO 35 and print a rough distance estimate based on signal strength.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm fires constantly with no flame | Sensitivity too high or IR interference | Reduce sensitivity trimmer; shield from incandescent lights or sunlight |
| Sensor never detects a candle flame | Sensitivity too low | Rotate trimmer clockwise to increase sensitivity |
| Buzzer beeps once and stops | `delay(500)` in the else branch overrides rapid re-triggering | Move the `delay` inside the alarm branch to maintain continuous alarm during flame |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [43 - ESP32 MQ-2 Gas Sensor Level Print](43-esp32-mq2-gas-sensor-level-print.md)
- [24 - ESP32 Limit Switch Detection Alert](24-esp32-limit-switch-detection-alert.md)
- [29 - ESP32 Vibration Sensor Latch Alarm](29-esp32-vibration-sensor-latch-alarm.md)
