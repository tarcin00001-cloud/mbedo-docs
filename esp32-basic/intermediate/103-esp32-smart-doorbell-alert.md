# 103 - ESP32 Smart Doorbell Alert

Build a motion-activated doorbell notifier that plays a "Ding-Dong" audio chime on a passive buzzer and displays welcome alerts on a 16x2 I2C LCD when presence is detected.

## Goal
Learn how to use PIR motion sensors to trigger complex multi-output routines, including structured audio chimes and LCD status changes.

## What You Will Build
A PIR sensor monitors motion on GPIO 4. A passive buzzer on GPIO 15 acts as the chime, and a 16x2 I2C LCD displays visitor messages. When motion is detected, the LCD prints "Visitor Detected!", the buzzer plays a two-tone "Ding-Dong" chime, and the system cools down for 6 seconds to prevent repeat triggers.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR501 PIR Sensor Module | `pir_sensor` | Yes | Yes |
| Passive Buzzer (Piezo Speaker) | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT | GPIO4 | Yellow | Presence detection input |
| PIR Sensor | VCC / GND | 5V / GND | Red / Black | Power rails (5V Vin pin) |
| Passive Buzzer | Positive (+) | GPIO15 | Blue | Audio chime signal |
| Passive Buzzer | Negative (−) | GND | Black | Ground reference |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Connect the PIR sensor VCC to the 5V (Vin) pin of the ESP32 to supply enough current for the internal PIR comparator circuit.

## Code
```cpp
// Smart Doorbell Alert
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN = 4;
const int BUZZER_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void playDingDong() {
  // "Ding" note (880 Hz - A5)
  tone(BUZZER_PIN, 880);
  delay(300);
  
  // "Dong" note (659 Hz - E5)
  tone(BUZZER_PIN, 659);
  delay(500);
  
  noTone(BUZZER_PIN);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  noTone(BUZZER_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Doorbell");
  lcd.setCursor(0, 1);
  lcd.print("System Active");
  
  delay(2000); // PIR sensor warm-up window
  lcd.clear();
}

void loop() {
  bool motion = (digitalRead(PIR_PIN) == HIGH);
  
  if (motion) {
    Serial.println("!! Motion detected: Visitor at door !!");
    
    // Update LCD
    lcd.setCursor(0, 0);
    lcd.print("Visitor Detected");
    lcd.setCursor(0, 1);
    lcd.print("Ding Dong!      ");
    
    // Play chime
    playDingDong();
    
    // Cooldown lockout period to prevent spamming
    delay(5000); 
    lcd.clear();
  } else {
    // Idle standby display
    lcd.setCursor(0, 0);
    lcd.print("Doorbell Standby");
    lcd.setCursor(0, 1);
    lcd.print("No Visitors     ");
    delay(200); // Poll PIR at 5Hz
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **PIR Sensor**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire PIR OUT to **GPIO4**, Buzzer to **GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Click the PIR sensor widget on the canvas. Watch the LCD print the visitor warning and the buzzer play the ding-dong chime.

## Expected Output
Serial Monitor:
```
!! Motion detected: Visitor at door !!
```

LCD Display (during visitor detection):
```
Visitor Detected
Ding Dong!
```

## Expected Canvas Behavior
* While the PIR sensor is idle, the LCD displays "Doorbell Standby".
* Activating the PIR widget immediately triggers the "Ding Dong" chime and changes the LCD to display "Visitor Detected".
* The system locks out and ignores new inputs for 5 seconds before returning to standby.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, 880)` | Generates the first note ("Ding") on the passive buzzer. |
| `tone(BUZZER_PIN, 659)` | Generates the second note ("Dong") on the passive buzzer. |
| `delay(5000)` | Cooldown lockout so visitors pressing or triggering multiple times don't sound the alarm repeatedly. |

## Hardware & Safety Concept: Doorbell Lockout Windows
Mechanical doorbells or alarm motion sensors include lockout window timers. If a visitor triggers a sensor multiple times, the chime logic ignores subsequent events within a 5–10 second range to avoid annoying occupants and prevent damage to mechanical bells or buzzers.

## Try This! (Challenges)
1. **Physical Doorbell Button**: Add a push button on GPIO 12 that also triggers the chime when pressed (traditional doorbell override).
2. **Visitor Counter**: Display the total number of visitor alerts recorded today on the LCD bottom row.
3. **Flashing Entry Light**: Add an LED on GPIO 5 that flashes during the chime alerts.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Doorbell triggers constantly | PIR sensor sensitivity is too high | Adjust the physical sensor trimmer to reduce range or avoid direct sunlight |
| Chime sounds flat or single-pitched | Active buzzer used instead of passive | Swap to a passive buzzer module |
| LCD does not clear | Delay issues | Ensure the `lcd.clear()` command is executed after the lockout delay completes |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 PIR Motion Sensor Alert](../beginner/36-esp32-pir-motion-sensor-alert.md)
- [70 - ESP32 Active Buzzer Chime Scale Generator](70-esp32-active-buzzer-chime-scale-generator.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
