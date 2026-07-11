# 141 - ESP32 Sounding Sentry Guard

Build a dual-stage security guard that integrates a PIR motion sensor to detect human presence and an HC-SR04 ultrasonic distance sensor to calculate range, sounding a buzzer siren and printing intruder alerts on an LCD screen.

## Goal
Learn how to implement dual-sensor security zoning, resolve compound input conditions, and manage visual and acoustic warning levels.

## What You Will Build
A PIR sensor is connected to GPIO 4, and an HC-SR04 to GPIO 12/13. A buzzer is on GPIO 15, and a 16x2 I2C LCD displays security statuses. If no motion is detected, the LCD shows "SYSTEM SECURE". If motion is detected but distance is > 100 cm, it displays "Warning: Approach". If motion is detected AND distance is <= 100 cm, it sounds the buzzer and prints "INTRUDER ALERT!".

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| HC-SR501 PIR Motion Sensor | `pir_sensor` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT | GPIO4 | Yellow | Presence detector input |
| PIR Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |
| HC-SR04 Sensor | Trig | GPIO12 | Orange | Sonar trigger |
| HC-SR04 Sensor | Echo | GPIO13 | Yellow | Sonar echo input |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power rails |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Alarm output |
| Active Buzzer | GND (−) | GND | Black | Ground reference |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |

> **Wiring tip:** Share the 5V Vin rail to power the LCD, PIR, and HC-SR04. Wire the active buzzer output to GPIO 15.

## Code
```cpp
// Sounding Sentry Guard
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN = 4;
const int TRIG_PIN = 12;
const int ECHO_PIN = 13;
const int BUZZER_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

const int SECURE_ZONE_CM = 100; // Trigger alarm if intruder is closer than this

float getDistance() {
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout
  if (duration == 0) return 999.0;
  return (duration * 0.0343) / 2.0;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Sentry Guard");
  lcd.setCursor(0, 1);
  lcd.print("Warming up...");
  
  delay(1500); // Wait for PIR sensor stabilization
  lcd.clear();
}

void loop() {
  bool motion = (digitalRead(PIR_PIN) == HIGH);
  float distance = getDistance();
  
  Serial.print("Motion: "); Serial.print(motion ? "YES" : "NO");
  Serial.print(" | Distance: "); Serial.print(distance, 1); Serial.println(" cm");
  
  if (motion) {
    // Stage 1: Intruder entered secure perimeter (distance <= 100 cm)
    if (distance <= SECURE_ZONE_CM) {
      Serial.println("!! ALARM: SECURE PERIMETER BREACHED !!");
      lcd.setCursor(0, 0);
      lcd.print("!! INTRUDER !!  ");
      lcd.setCursor(0, 1);
      lcd.print("Dist: "); lcd.print(distance, 0); lcd.print("cm      ");
      
      // Pulse alarm siren
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      delay(150);
    } 
    // Stage 2: Motion detected but outside secure boundary
    else {
      Serial.println("Warning: Motion detected at distance.");
      lcd.setCursor(0, 0);
      lcd.print("Warning: Move   ");
      lcd.setCursor(0, 1);
      lcd.print("Scan: Approaching");
      digitalWrite(BUZZER_PIN, LOW);
      delay(200);
    }
  } 
  // Stage 3: Secure standby
  else {
    lcd.setCursor(0, 0);
    lcd.print("Sentry: ACTIVE  ");
    lcd.setCursor(0, 1);
    lcd.print("Status: SECURE  ");
    digitalWrite(BUZZER_PIN, LOW);
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04**, **PIR Sensor**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire PIR to **GPIO4**, Trig/Echo to **GPIO12/GPIO13**, Buzzer to **GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the HC-SR04 distance widget to 150 cm and trigger PIR motion. Observe the warning message.
5. Slide the distance to 50 cm and trigger motion. Watch the LCD print alert and the buzzer sound.

## Expected Output
Serial Monitor:
```
Sentry Guard Warming up...
Motion: NO | Distance: 154.2 cm
Motion: YES | Distance: 120.0 cm
Warning: Motion detected at distance.
Motion: YES | Distance: 60.5 cm
!! ALARM: SECURE PERIMETER BREACHED !!
```

LCD Display (breached):
```
!! INTRUDER !!
Dist: 60cm
```

## Expected Canvas Behavior
* No motion: LCD shows "Sentry: ACTIVE / Status: SECURE".
* PIR triggered + Distance > 100 cm: LCD shows "Warning: Move / Scan: Approaching".
* PIR triggered + Distance <= 100 cm: LCD flashes warning, buzzer pulses green (ON).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `motion` | Polls the PIR sensor state. |
| `distance <= SECURE_ZONE_CM` | Evaluates if the object has breached the close-range threshold. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives the siren on breaches. |

## Hardware & Safety Concept: Multi-stage Defense and False Alarm Mitigation
PIR sensors are highly sensitive to thermal shifts (heat vectors), meaning wind, small animals, or heating vents can trigger false alarms. By adding a secondary time-of-flight distance check, security systems verify that a physical object is actually approaching a specific target zone before activating the acoustic siren.

## Try This! (Challenges)
1. **Intruder counter log**: Count the number of breaches and log them to memory, displaying the count on the LCD.
2. **Camera trigger relay**: Add a relay output on GPIO 14 that simulates triggering a security camera recording when the alarm state occurs.
3. **Mute switch toggle**: Wire a switch on GPIO 25 that disables the buzzer (silent alarm mode).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers when no motion is present | PIR sensor adjusting time | PIR sensors require 30–60 seconds of quiet calibration at boot |
| Distance reads static 999.0 | Sonar pin miswired | Check trigger/echo wiring on GPIO 12 and 13 |
| Buzzer click is weak | Weak current from ESP32 | Ensure active buzzer is driven correctly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [75 - ESP32 HC-SR04 Proximity Sensor Serial Logs](../intermediate/75-esp32-hcsr04-proximity-sensor-serial-logs.md)
- [108 - ESP32 Intrusion Detector Alarm](108-esp32-intrusion-detector-alarm.md)
