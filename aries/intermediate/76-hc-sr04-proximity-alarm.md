# 76 - HC-SR04 Proximity Alarm

Build a safety proximity alert system that monitors physical distance using an HC-SR04 ultrasonic sensor and triggers a buzzer if an object gets within 20 cm.

## Goal
Learn how to use distance parameters as conditional triggers inside the main loop, activating audio indicators when objects breach a safety perimeter.

## What You Will Build
An HC-SR04 sensor measures distance (Trig on GPIO 14, Echo on GPIO 15). An Active Buzzer connects to GPIO 12. If an object is detected closer than 20.0 cm, the buzzer beeps rapidly to alert the user.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Power input |
| HC-SR04 Sensor | Trig | GPIO 14 | Orange | Trigger control line |
| HC-SR04 Sensor | Echo | GPIO 15 | Yellow | Echo signal line |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |
| Active Buzzer | Positive (+) | GPIO 12 | Red | Buzzer control line |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |

> **Wiring tip:** Share the GND connections on a breadboard. Be careful not to confuse the buzzer pin (GPIO 12) with the sensor pins.

## Code
```cpp
// HC-SR04 Proximity Alarm
const int TRIG_PIN = 14;
const int ECHO_PIN = 15;
const int BUZZER_PIN = 12;

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("Proximity Alarm System active.");
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
  Serial.print(distance, 1);
  Serial.println(" cm");
  
  // Trigger alarm if distance is less than 20 cm
  if (duration > 0 && distance < 20.0) {
    Serial.println("!! BREACH ALERT: Object too close !!");
    
    // Rapid beep sequence
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    delay(100);
  } else {
    digitalWrite(BUZZER_PIN, LOW);
    delay(400); // Wait between normal scans
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-SR04 Sensor**, and **Active Buzzer** components onto the canvas.
2. Wire the sensor: **Trig** to **GPIO 14**, **Echo** to **GPIO 15**, **VCC** to **5V**, and **GND** to **GND**.
3. Wire the buzzer: **Positive (+)** to **GPIO 12** and **Negative (-)** to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Move the distance slider on the HC-SR04 widget below 20 cm and listen to the buzzer.

## Expected Output
Serial Monitor:
```
Proximity Alarm System active.
Distance: 34.5 cm
Distance: 15.2 cm
!! BREACH ALERT: Object too close !!
```

## Expected Canvas Behavior
* The active buzzer sounds a pulsing warning beep when the distance slider on the HC-SR04 component goes below 20 cm.
* Moving the slider back beyond 20 cm turns the buzzer off.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Configures GPIO 12 as buzzer output. |
| `distance < 20.0` | Conditional check evaluating if an object is closer than 20 cm. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GPIO 12 HIGH to turn on the buzzer. |
| `delay(100)` | Keeps the buzzer ON for 100 milliseconds for a distinct pulse length. |

## Hardware & Safety Concept
* **Collision Detection Zones**: Industrial automated guided vehicles (AGVs) and car parking sensors use ultrasonic distance measurements. They define multiple zones: warning zone (buzzer beeps slowly) and stop zone (buzzer sounds continuously and motors stop). This prevents collisions by providing early warning alerts based on raw physical distance.

## Try This! (Challenges)
1. **Dynamic Beeping Rate**: Modify the code so the beep rate increases as the obstacle gets closer (e.g. shorter delays when distance is very small).
2. **Visual Warning LED**: Connect a Warning LED to GPIO 15. Light the LED when the distance is under 40 cm, and trigger the buzzer when it drops under 20 cm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Incorrect buzzer GPIO pin | Verify the buzzer positive lead is connected to GPIO 12. |
| Alarm triggers continuously | Zero distance reading | Check wiring of Echo and Trig; a broken connection reads 0 which is < 20. |
| Delay in alarm trigger | Loop delays are too long | Keep code execution delays to a minimum to ensure responsiveness. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - HC-SR04 Proximity Sensor Serial Logs](75-hc-sr04-proximity-sensor-serial.md)
- [77 - HC-SR04 Distance Bar Graph LCD](77-hc-sr04-distance-bar-graph-lcd.md)
- [13 - Buzzer Warning Pulse](../beginner/13-buzzer-warning-pulse.md)
