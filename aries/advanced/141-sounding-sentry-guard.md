# 141 - Sounding Sentry Guard

Combine a PIR motion sensor, an HC-SR04 ultrasonic distance sensor, and an active buzzer to construct a multi-stage intrusion detection system that updates status alerts on a 16x2 LCD using the VEGA ARIES v3 board.

## Goal
Learn how to read high-frequency timing sensors (ultrasonic distance), process digital state change indicators (PIR motion), execute immediate actuator responses (buzzer alarms), and output structured warnings to an LCD screen.

## What You Will Build
An intrusion sentinel. The system monitors a designated zone:
- **PIR Motion Tracking**: Detects heat signatures of moving objects.
- **Ultrasonic Range Finder**: Measures the distance of objects in front of the sensor.
- **Intrusion Alarm**: If motion is detected OR an object is detected within 50 cm, the system sounds a warning siren on the buzzer (GPIO 12) and prints `ALERT: Intrusion` on the LCD. Otherwise, the system displays `Status: Secure`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `hc_sr04` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Primary 5V power |
| HC-SR04 Sensor | Trig | GPIO 14 | Orange | Trigger pulse output |
| HC-SR04 Sensor | Echo | GPIO 15 | Yellow | Echo return input |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | VCC | 3V3 | Red | Sensor power (3.3V) |
| PIR Sensor | OUT | GPIO 17 | Purple | Motion digital input (Shared) |
| PIR Sensor | GND | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO 12 | Green | Buzzer activation output |
| Active Buzzer | GND (-) | GND | Grey | Ground reference |
| I2C LCD Display | VCC | 5V | Red | Primary 5V power |
| I2C LCD Display | GND | GND | Black | Ground reference |
| I2C LCD Display | SDA | SDA0 (GP17) | Blue | I2C Data line (Shared GP17) |
| I2C LCD Display | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The PIR Sensor output connects to GPIO 17. Note that the I2C0 SDA line also shares GP17. The interpreter and hardware logic will coordinate this sharing in the simulation.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int trigPin = 14;
const int echoPin = 15;
const int pirPin = 17;
const int buzzerPin = 12;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);
  delay(1000);

  pinMode(trigPin, OUTPUT);
  pinMode(echoPin, INPUT);
  pinMode(pirPin, INPUT);
  pinMode(buzzerPin, OUTPUT);

  // Initialize Buzzer to OFF
  digitalWrite(buzzerPin, LOW);

  Wire.begin();
  lcd.init();
  lcd.backlight();
  lcd.print("Sentry Guard ON");
  delay(1500);
  lcd.clear();
}

void loop() {
  // Trigger the HC-SR04 sensor
  digitalWrite(trigPin, LOW);
  delayMicroseconds(2);
  digitalWrite(trigPin, HIGH);
  delayMicroseconds(10);
  digitalWrite(trigPin, LOW);

  // Measure the Echo duration in microseconds
  long duration = pulseIn(echoPin, HIGH);
  
  // Calculate distance in cm (Speed of sound is 340 m/s or 0.034 cm/us)
  int distance = duration * 0.034 / 2;

  // Read the PIR sensor state
  int motionDetected = digitalRead(pirPin);

  lcd.setCursor(0, 0);

  // Alarm Condition: Motion detected OR target closer than 50 cm
  if (motionDetected == HIGH || (distance > 0 && distance < 50)) {
    digitalWrite(buzzerPin, HIGH); // Alarm ON
    
    lcd.print("ALERT: INTRUDER!");
    lcd.setCursor(0, 1);
    lcd.print("D: ");
    lcd.print(distance);
    lcd.print("cm | PIR: ACT");

    Serial.print("ALARM - Intruder Detected! Distance: ");
    Serial.print(distance);
    Serial.print(" cm | Motion: ");
    Serial.println(motionDetected);
  } else {
    digitalWrite(buzzerPin, LOW); // Alarm OFF
    
    lcd.print("STATUS: SECURE  ");
    lcd.setCursor(0, 1);
    lcd.print("D: ");
    if (distance > 400 || distance <= 0) {
      lcd.print("Out of range ");
    } else {
      lcd.print(distance);
      lcd.print("cm        ");
    }

    Serial.print("Secure - Distance: ");
    Serial.print(distance);
    Serial.print(" cm | Motion: ");
    Serial.println(motionDetected);
  }

  delay(500); // Sample twice per second
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **HC-SR04 Sensor**, **PIR Motion Sensor**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect the HC-SR04: **VCC** to **5V**, **GND** to **GND**, **Trig** to **GPIO 14**, and **Echo** to **GPIO 15**.
3. Connect the PIR: **VCC** to **3V3**, **GND** to **GND**, and **OUT** to **GPIO 17**.
4. Connect the Buzzer: **VCC** to **GPIO 12**, and **GND** to **GND**.
5. Connect the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
6. Paste the code, select **Interpreted Mode**, and click **Run**.
7. Click the PIR or HC-SR04 widgets to trigger alarms.

## Expected Output
Serial Monitor:
```
System Initialized.
Secure - Distance: 120 cm | Motion: 0
ALARM - Intruder Detected! Distance: 35 cm | Motion: 0
ALARM - Intruder Detected! Distance: 120 cm | Motion: 1
```

## Expected Canvas Behavior
* When the PIR slider indicates motion or the HC-SR04 distance drops below 50 cm, the buzzer widget pulses red (active).
* The LCD immediately changes from `STATUS: SECURE` to `ALERT: INTRUDER!` with updated metrics.
* Readings and alarms reset as soon as targets leave the sensor fields.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pulseIn(echoPin, HIGH)` | Reads the width of the incoming HIGH logic pulse on pin 15, matching flight time. |
| `duration * 0.034 / 2` | Mathematical conversion factors mapping sound time-of-flight to distance in cm. |
| `digitalRead(pirPin)` | Reads the latch state from the PIR sensor's internal comparator circuit. |
| `digitalWrite(buzzerPin, HIGH)` | Applies 3.3V logic to turn on the active buzzer alarm. |
| `lcd.print("ALERT: INTRUDER!")` | Updates the top display line to notify of an ongoing security breach. |

## Hardware & Safety Concept
* **Time-Of-Flight Principles**: The HC-SR04 sends eight 40kHz ultrasonic pulses. If an obstacle reflects these pulses, the echo pin goes HIGH for the exact duration the wave travels to the target and back.
* **PIR Blind Period**: Most PIR sensors have an adjustable delay (typically 3s to 5min) and a brief lock-out period (1.2s to 3s) during which they cannot detect motion. Account for these pauses in your design.

## Try This! (Challenges)
1. **Pulse Alert Tone**: Instead of holding the buzzer HIGH continuously, toggle the buzzer pin ON and OFF every 100 milliseconds to create a chirping siren pattern.
2. **Safe Distance Threshold**: Expand the logic so that if the distance is between 50 cm and 100 cm, the LCD prints `STATUS: WARNING` and the buzzer chirps slowly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always reads 0 | Echo pin connection loose | Check the wiring of GP15 (Echo) and ensure the trigger pulse is fired correctly. |
| Buzzer triggers randomly | PIR sensitivity issues | Adjust the sensitivity potentiometer on the PIR sensor board to avoid false triggers. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [140 - Multi-Sensor Air Quality HUD](140-multi-sensor-air-quality-hud.md) (Previous project)
- [142 - Tilt Vault Alarm](142-tilt-vault-alarm.md) (Next project)
