# 120 - Autonomous Guard

Build a dual-stage security perimeter controller using an ultrasonic distance sensor (HC-SR04) and a PIR motion sensor. PIR detects presence to trigger a warning stage, while ultrasonic proximity measurement triggers a high-severity alarm stage.

## Goal
Learn how to implement a nested, dual-stage alert logic system using digital motion detection and analog-equivalent timing measurements, coordinating multiple outputs (active buzzer, warning LED, LCD) in real-time.

## What You Will Build
An automated security sentry:
- **Stage 1 (PIR Warning)**: If the PIR sensor detects motion, a yellow/red warning LED flashes, and the LCD displays "Intruder Warning".
- **Stage 2 (Ultrasonic Alarm)**: If the intruder approaches closer than 30 cm, the buzzer rings continuously, the LED remains solid, and the LCD displays "INTRUDER ALERT!".
- **System Normal**: If no motion is detected and the distance is clear, the LCD displays "Perimeter Secure".

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| PIR Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Power supply |
| HC-SR04 Sensor | TRIG | D3 | Trigger pulse pin |
| HC-SR04 Sensor | ECHO | D4 | Echo pulse return pin |
| HC-SR04 Sensor | GND | GND | Ground reference |
| PIR Sensor | OUT | D2 | Digital motion output |
| LED | Anode | D5 | Warning indicator LED |
| Active Buzzer | VCC | D8 | Alarm feedback pin |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN     = 2;
const int TRIG_PIN    = 3;
const int ECHO_PIN    = 4;
const int LED_PIN     = 5;
const int BUZZER_PIN  = 8;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Sentry Active");
  delay(1500);
  lcd.clear();

  Serial.println("Autonomous Sentry Online");
}

void loop() {
  // Read PIR Motion
  int motion = digitalRead(PIR_PIN);

  // Trigger HC-SR04 to measure distance
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  float distance = duration * 0.034 / 2.0;

  Serial.print("Motion: ");
  Serial.print(motion ? "DETECTED" : "NONE");
  Serial.print(" | Distance: ");
  Serial.print(distance);
  Serial.println(" cm");

  // Dual-stage alert logic
  if (motion == HIGH) {
    if (distance > 0 && distance < 30) {
      // Stage 2: Critical proximity intrusion
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(LED_PIN, HIGH);
      
      lcd.setCursor(0, 0);
      lcd.print("INTRUDER ALERT! ");
      lcd.setCursor(0, 1);
      lcd.print("RANGE: ");
      lcd.print(distance, 0);
      lcd.print(" cm       ");
    } else {
      // Stage 1: Perimeter breach warning
      digitalWrite(BUZZER_PIN, LOW);
      
      // Flash LED warning
      digitalWrite(LED_PIN, HIGH);
      delay(150);
      digitalWrite(LED_PIN, LOW);

      lcd.setCursor(0, 0);
      lcd.print("Intruder Warning");
      lcd.setCursor(0, 1);
      lcd.print("Motion Detected ");
    }
  } else {
    // System normal
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(LED_PIN, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Perimeter Secure");
    lcd.setCursor(0, 1);
    lcd.print("Status: Normal  ");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-SR04 Ultrasonic**, **PIR Sensor**, **LED**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect HC-SR04: **VCC** to **5V**, **TRIG** to **D3**, **ECHO** to **D4**, **GND** to **GND**.
3. Connect PIR: **OUT** to **D2**, **VCC** to **5V**, **GND** to **GND**.
4. Connect LED: **Anode** to **D5**, **Cathode** to **GND**.
5. Connect Buzzer: **VCC** to **D8**, **GND** to **GND**.
6. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
7. Paste code, select the interpreted mode, and click **Run**.
8. Modify both PIR output and distance sliders to test both alarm stages.

## Expected Output

Terminal:
```
Autonomous Sentry Online
Motion: NONE | Distance: 150.20 cm
Motion: DETECTED | Distance: 148.50 cm
Motion: DETECTED | Distance: 25.40 cm
```

LCD Display:
```
INTRUDER ALERT! 
RANGE: 25 cm       
```

## Expected Canvas Behavior
| PIR State (D2) | Distance (HC-SR04) | LED (D5) | Buzzer (D8) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- | --- |
| LOW | > 30 cm | OFF | OFF | `Perimeter Secure` | `Status: Normal` |
| HIGH | 80 cm | Flashing | OFF | `Intruder Warning` | `Motion Detected` |
| HIGH | 20 cm | Solid ON | ON | `INTRUDER ALERT!` | `RANGE: 20 cm` |
| LOW | 15 cm | OFF | OFF | `Perimeter Secure` | `Status: Normal` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pulseIn(ECHO_PIN, HIGH)` | Reads the duration in microseconds for the ultrasonic return pulse to go HIGH and back to LOW. |
| `duration * 0.034 / 2.0` | Converts time duration to distance in centimeters based on the speed of sound (343 m/s). |
| `motion == HIGH && distance < 30` | Validates a compound state where motion presence is confirmed within close range. |

## Hardware & Safety Concept: Sensor Fusion
Sensor Fusion is the practice of combining data from multiple sensors to achieve more accurate assessments of external states. Here, a PIR sensor provides a wide-angle presence trigger, while the ultrasonic sensor provides high-accuracy depth metrics. This reduces false alarms (e.g. motion from a far-off object won't trigger the siren).

## Try This! (Challenges)
1. **Intruder Speed Estimation**: Record the difference in distance between two cycles. If the intruder's distance decreases faster than 20 cm/cycle, trigger the siren immediately even if they are farther than 30 cm.
2. **Panic Reset Button**: Add a push button to pin D7. Once Stage 2 is triggered, require a button press to silence the alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance always reads 0 | ECHO/TRIG pins swapped | Verify that D3 is wired to TRIG, and D4 is wired to ECHO. |
| Alarm oscillates rapidly | Delay in main loop too short | Increase the delay at the end of the loop to stabilize sensor reads. |

## Mode Notes
The HC-SR04 pulseIn pattern and PIR state checking are supported by MbedO interpreted mode.

## Related Projects
- [31 - Motion Alarm](../intermediate/31-motion-alarm.md)
- [56 - Parking Beeper](../intermediate/56-parking-beeper.md)
- [98 - Proximity Guard](../advanced/98-proximity-guard.md)
