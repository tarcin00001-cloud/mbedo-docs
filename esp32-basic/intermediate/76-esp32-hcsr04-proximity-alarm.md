# 76 - ESP32 HC-SR04 Proximity Alarm

Build a parking sensor or collision avoidance alarm that triggers a visual LED indicator and a pulsing buzzer as an obstacle gets closer.

## Goal
Learn how to apply nested threshold conditions to distance measurements to build a multi-stage proximity alert system.

## What You Will Build
An HC-SR04 sensor reads proximity on GPIO 5 (Trig) and GPIO 18 (Echo). If an object is detected within 30 cm, a warning LED on GPIO 5 (alarm LED on GPIO 4) flashes. If it gets closer than 15 cm, a buzzer on GPIO 15 sounds continuously to alert the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | Trig | GPIO5 | Orange | Trigger output |
| HC-SR04 Sensor | Echo | GPIO18 | Yellow | Echo input |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power |
| LED (Red) | Anode (+) | GPIO4 via 330 Ω | Orange | Warning indicator |
| LED (Red) | Cathode (−) | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |

> **Wiring tip:** Since GPIO 5 is used for the ultrasonic Trig pin in this setup, connect the warning LED to **GPIO 4** instead. Verify all ground lines are tied together.

## Code
```cpp
// HC-SR04 Proximity Alarm
const int TRIG_PIN = 5;
const int ECHO_PIN = 18;
const int LED_PIN = 4;
const int BUZZER_PIN = 15;

// Distance thresholds in centimeters
const float WARN_DIST = 30.0;
const float STOP_DIST = 15.0;

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("Proximity Alarm Station Ready.");
}

void loop() {
  // Trigger sensor pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  float distance = (duration * 0.0343) / 2.0;
  
  Serial.print("Distance: ");
  if (duration == 0 || distance > 400) {
    Serial.println("Out of range");
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    Serial.print(distance, 1);
    Serial.println(" cm");
    
    // Critical Collision Zone (0 to 15cm)
    if (distance <= STOP_DIST) {
      Serial.println("!! CRITICAL STOP !!");
      digitalWrite(LED_PIN, HIGH);
      digitalWrite(BUZZER_PIN, HIGH); // Continuous alarm
    } 
    // Warning Zone (15.1cm to 30cm)
    else if (distance <= WARN_DIST) {
      Serial.println("Warning: Object Close");
      // Fast pulsing warning
      digitalWrite(LED_PIN, HIGH);
      digitalWrite(BUZZER_PIN, HIGH);
      delay(100);
      digitalWrite(LED_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
      delay(100);
    } 
    // Safe Zone (> 30cm)
    else {
      digitalWrite(LED_PIN, LOW);
      digitalWrite(BUZZER_PIN, LOW);
    }
  }
  
  delay(200); // 5Hz refresh cycle
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, **LED**, and **Buzzer** onto the canvas.
2. Wire Trig to **GPIO5**, Echo to **GPIO18**, LED to **GPIO4**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Slide the virtual sensor distance to different levels and watch the alarm behavior change.

## Expected Output
Serial Monitor:
```
Proximity Alarm Station Ready.
Distance: 45.3 cm
Distance: 25.1 cm
Warning: Object Close
Distance: 10.4 cm
!! CRITICAL STOP !!
```

## Expected Canvas Behavior
* Outside 30 cm, both LED and buzzer remain inactive.
* Between 15 cm and 30 cm, the red LED and buzzer pulse rapidly on and off.
* Below 15 cm, the LED and buzzer stay ON continuously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `distance <= STOP_DIST` | Triggers the critical stage if an obstacle is within 15 cm. |
| `distance <= WARN_DIST` | Triggers the pulsing warning zone if an obstacle is within 30 cm (but greater than 15 cm). |
| `digitalWrite(BUZZER_PIN, HIGH)` | Sounds the active buzzer alert. |

## Hardware & Safety Concept: Multi-stage Range Warning Systems
Parking sensors use multi-stage alerts to help drivers gauge distance. As the vehicle approaches an obstacle, the frequency of beeping increases, becoming a solid tone when the vehicle reaches the minimum safe distance. This physical warning pattern is implemented in code using cascaded `if-else` branches.

## Try This! (Challenges)
1. **Dynamic Beeping Tempo**: Make the alarm beeping interval decrease smoothly as distance decreases, using mathematical formulas instead of fixed step logic.
2. **Reverse Warning light**: Add a yellow warning LED on GPIO 14 that lights up when entering the warning zone, leaving the red LED only for the stop zone.
3. **Mute Switch**: Implement a toggle switch on GPIO 23 that turns off the buzzer sounds but keeps the LED alerts active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm pulses even when no obstacle is near | Ghost echoes or reflections from the floor | Angle the sensor upwards slightly or clear the field of view |
| Buzzer does not sound in stop zone | Missing pin definitions | Ensure `pinMode(BUZZER_PIN, OUTPUT)` is declared in setup |
| Measurements jump between zero and actual distances | Echo pulse missed | Verify the ground wire connection between sensor and ESP32 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - ESP32 HC-SR04 Proximity Sensor Serial Logs](75-esp32-hcsr04-proximity-sensor-serial-logs.md)
- [77 - ESP32 HC-SR04 Distance Bar Graph LCD](77-esp32-hcsr04-distance-bar-graph-lcd.md)
- [120 - ESP32 Autonomous Guard](../expert/120-autonomous-guard.md)
