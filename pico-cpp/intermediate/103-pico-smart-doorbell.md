# 103 - Pico Smart Doorbell

Build a smart doorbell system that plays a chime and displays visitor status on an LCD when motion is detected.

## Goal
Learn how to use digital motion sensor triggers (PIR) to display greeting messages on LCDs and play two-tone chimes.

## What You Will Build
A visitor notifier console:
- **PIR Motion Sensor (GP16)**: Detects approaching visitors.
- **16x2 I2C LCD (GP4, GP5)**: Displays "Visitor Detected" when motion is detected, and "Monitoring..." when idle.
- **Passive Buzzer (GP14)**: Plays a happy two-tone chime (Ding-Dong) once when motion is first detected.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply |
| PIR Sensor | OUT | GP16 | Motion sense pin |
| PIR Sensor | GND | GND | Ground reference |
| Passive Buzzer | VCC (+) | GP14 | Chime output |
| Passive Buzzer | GND (-) | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PIR_PIN    = 16;
const int BUZZER_PIN = 14;

LiquidCrystal_I2C lcd(0x27, 16, 2);

bool lastMotionState = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Doorbell");
  lcd.setCursor(0, 1);
  lcd.print("Monitoring...   ");
}

void loop() {
  int motionDetected = digitalRead(PIR_PIN);

  // Transition check: motion changed from idle to active
  if (motionDetected == HIGH && lastMotionState == false) {
    lcd.setCursor(0, 1);
    lcd.print("Visitor Detected");
    
    // Play Ding-Dong chime
    tone(BUZZER_PIN, 523, 300); // Ding (C5)
    delay(350);
    tone(BUZZER_PIN, 392, 400); // Dong (G4)
    delay(450);
    
    lastMotionState = true;
  } 
  // Motion cleared
  else if (motionDetected == LOW && lastMotionState == true) {
    lcd.setCursor(0, 1);
    lcd.print("Monitoring...   ");
    lastMotionState = false;
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **PIR Motion Sensor**, **Passive Buzzer**, and **I2C LCD** onto the canvas.
2. Connect PIR OUT to **GP16**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Click the PIR Sensor on canvas to simulate a visitor, and watch the LCD update and hear the Ding-Dong chime.

## Expected Output

Terminal:
```
Simulation active. Smart doorbell active.
```

## Expected Canvas Behavior
* Normal state: LCD displays `Monitoring...`.
* Motion Scanned (GP16 HIGH): LCD changes to `Visitor Detected`, buzzer plays "Ding-Dong" chime.
* Motion cleared: LCD reverts to `Monitoring...`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `motionDetected == HIGH && lastMotionState == false` | Trigger verification ensuring the chime only plays once per motion event rather than playing continuously while the visitor stands at the door. |

## Hardware & Safety Concept: Visitor Privacy Zones
Smart doorbells use PIR motion sensors to alert homeowners of approaching visitors. To prevent false alerts from cars on nearby streets or wind moving trees, real doorbell lenses are shaped to limit the vertical field of view, focusing on visitors within 2-3 meters of the door.

## Try This! (Challenges)
1. **Intruder Warn LED**: Add a Red LED on GP15 that flashes if the PIR sensor detects motion for more than 5 seconds (indicating loitering).
2. **Custom Chime**: Rewrite the frequency notes to play a Westminster Quarters melody.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Chime plays repeatedly while person is static | Cooldown dial set high | Standard PIR sensors hold their HIGH state for several seconds after detecting motion. Ensure your code uses state transition checks (`lastMotionState`) to prevent repeat chime triggers. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [36 - Pico PIR Motion](../../beginner/36-pico-pir-motion.md)
- [70 - Pico Buzzer Scale](../intermediate/70-pico-buzzer-scale.md)
