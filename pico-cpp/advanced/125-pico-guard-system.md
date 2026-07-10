# 125 - Pico Guard System

Build an integrated security guard monitor station that logs motion detection events and targets intruder proximity.

## Goal
Learn how to combine digital motion sensors (PIR) and ultrasonic distance sensors (HC-SR04) to build multi-stage intruder alert logic.

## What You Will Build
An integrated perimeter security console:
- **PIR Motion Sensor (GP16)**: Scans for general entry movement.
- **HC-SR04 Ultrasonic Sensor (Trig GP14, Echo GP15)**: Activates when motion is detected to measure the intruder's proximity distance.
- **Active Buzzer (GP10)**: Sounds a pulsing alarm if the intruder gets closer than 50 cm.
- **Red LED (GP13)**: Illuminates during proximity alarm phases.
- **16x2 I2C LCD (GP4, GP5)**: Displays status readouts (e.g. "Intruder Alert!" and "Dist: 35 cm").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | OUT | GP16 | Movement detection |
| HC-SR04 | Trig | GP14 | Distance trigger |
| HC-SR04 | Echo | GP15 | Distance echo (with divider) |
| Active Buzzer | VCC (+) | GP10 | Acoustic indicator |
| Red LED | Anode | GP13 | Visual indicator |
| I2C LCD | SDA / SCL | GP4 / GP5 | Status screen |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN    = 16;
const int TRIG_PIN   = 14;
const int ECHO_PIN   = 15;
const int BUZZ_PIN   = 10;
const int LED_PIN    = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(PIR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);

  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);
  digitalWrite(LED_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Guard Station");
  lcd.setCursor(0, 1);
  lcd.print("System Secured  ");
  delay(1000);
}

void loop() {
  int motion = digitalRead(PIR_PIN);

  if (motion == HIGH) {
    // 1. Measure distance of intruder
    digitalWrite(TRIG_PIN, HIGH);
    delayMicroseconds(10);
    digitalWrite(TRIG_PIN, LOW);

    long duration = pulseIn(ECHO_PIN, HIGH);
    int distance = duration * 0.0343 / 2;

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("INTRUDER ALERT!");

    lcd.setCursor(0, 1);
    if (distance > 0 && distance < 50) {
      // Critical proximity alarm (intrusion within 50 cm)
      lcd.print("Dist: ");
      lcd.print(distance);
      lcd.print("cm CLOSE!");

      digitalWrite(LED_PIN, HIGH);
      
      // Fast alarm beep
      digitalWrite(BUZZ_PIN, HIGH);
      delay(80);
      digitalWrite(BUZZ_PIN, LOW);
      delay(80);
    } else {
      // Warning phase (intruder is far away)
      lcd.print("Dist: ");
      lcd.print(distance);
      lcd.print("cm warning");
      
      digitalWrite(LED_PIN, LOW);
      digitalWrite(BUZZ_PIN, LOW);
    }
  } else {
    // System safe
    digitalWrite(BUZZ_PIN, LOW);
    digitalWrite(LED_PIN, LOW);

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Guard Station");
    lcd.setCursor(0, 1);
    lcd.print("System Secured  ");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **PIR Sensor**, **HC-SR04 Sensor**, **Active Buzzer**, **Red LED**, and **I2C LCD** onto the canvas.
2. Connect PIR OUT to **GP16**, HC-SR04 to **GP14/GP15**, Buzzer to **GP10**, LED to **GP13**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Trigger the PIR sensor first to start range monitoring, then slide the distance sensor to trigger alarms.

## Expected Output

Terminal:
```
Simulation active. Perimeter guard station monitoring active.
```

## Expected Canvas Behavior
* No Motion: LCD reads `System Secured`, LED and Buzzer are OFF.
* Motion + Far Distance (> 50 cm): LCD reads `INTRUDER ALERT!` / `Dist: 120cm warning`. LED/Buzzer OFF.
* Motion + Close Distance (< 50 cm): LCD reads `INTRUDER ALERT!` / `Dist: 35cm CLOSE!`. LED turns ON, Buzzer beeps rapidly.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motion == HIGH` | Condition check that activates the proximity tracking subsystem only when motion is detected. |

## Hardware & Safety Concept: Multi-Sensor Perimeter Security
High-security systems use multi-sensor networks (such as combining PIR and ultrasonic sensors) to filter out false alerts. PIR sensors are sensitive to changes in infrared heat (such as a warm body), but cannot measure distance. Ultrasonic sensors measure distance precisely, but consume more power. Activating the rangefinder only when motion is detected saves energy and improves detection reliability.

## Try This! (Challenges)
1. **Latching Security Alert**: If critical proximity is triggered, lock the alarm state ON until a secret override button (GP12) is held down.
2. **Dynamic Alert rate**: Make the buzzer pulse faster as the intruder gets closer.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm beeps constantly even when quiet | PIR sensor cooldown period | PIR sensors stay active for a few seconds after motion stops. Ensure you adjust the cooldown dial on the sensor module. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [76 - Pico Ultrasonic Alarm](../intermediate/76-pico-ultrasonic-alarm.md)
- [103 - Pico Smart Doorbell](../intermediate/103-pico-smart-doorbell.md)
- [121 - Pico Fire System](121-pico-fire-system.md)
