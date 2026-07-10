# 101 - Pico Automatic Gate

Build an automated toll gate barrier that opens a servo arm when an approaching vehicle is detected by an IR proximity sensor.

## Goal
Learn how to use digital input sensor triggers (IR obstacle sensor) to control servo angles and display status messages on an LCD.

## What You Will Build
An automatic parking gate:
- **IR Obstacle Sensor (GP16)**: Reads `LOW` when a vehicle approaches the gate.
- **Servo Motor (GP10)**: Rotates to 90° (Gate open) when a vehicle is detected, pauses for 3 seconds, and returns to 0° (Gate closed).
- **16x2 I2C LCD (GP4, GP5)**: Displays gate status (e.g. "Gate: Open" or "Gate: Closed").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| IR Obstacle Sensor | `button` | Yes (represented by switch button) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 3.3V | Power supply |
| IR Sensor | OUT | GP16 | Vehicle detect signal |
| IR Sensor | GND | GND | Ground return |
| Servo Motor | Signal | GP10 | Gate arm control |
| Servo Motor | Power | VBUS (5V) | Power supply |
| Servo Motor | Ground | GND | Ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <LiquidCrystal_I2C.h>

const int IR_PIN    = 16;
const int SERVO_PIN = 10;

Servo gateServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(IR_PIN, INPUT);
  gateServo.attach(SERVO_PIN);
  gateServo.write(0); // Start closed
  
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Smart Toll Gate");
  lcd.setCursor(0, 1);
  lcd.print("Gate: Closed    ");
}

void loop() {
  int vehicleDetected = digitalRead(IR_PIN);

  // Active-LOW check: IR sensor pulls OUT pin LOW when vehicle blocks beam
  if (vehicleDetected == LOW) {
    lcd.setCursor(0, 1);
    lcd.print("Gate: Open      ");
    gateServo.write(90); // Open barrier arm
    
    delay(3000); // Keep gate open for 3 seconds
    
    lcd.setCursor(0, 1);
    lcd.print("Gate: Closed    ");
    gateServo.write(0); // Close barrier arm
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **IR Obstacle Sensor** (represented by switch button), **Servo**, and **I2C LCD** onto the canvas.
2. Connect IR Sensor OUT to **GP16**, Servo to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Simulate a vehicle arrival by closing the switch contact representing the IR sensor on canvas, and watch the gate open.

## Expected Output

Terminal:
```
Simulation active. Automated toll gate controller online.
```

## Expected Canvas Behavior
* Normal state: LCD displays `Gate: Closed`, Servo at 0°.
* Vehicle Scanned (GP16 LOW): LCD changes to `Gate: Open`, Servo rotates to 90°. After 3 seconds, Servo returns to 0° and LCD reverts to `Gate: Closed`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `gateServo.write(90)` | Rotates the gate arm to 90 degrees to clear the path for the vehicle. |

## Hardware & Safety Concept: Obstacle Safety Loops
Real toll gates or garage doors must prevent the barrier arm from closing on top of a vehicle. To prevent accidents, a safety loop (usually an IR photobeam or loop detector wire under the pavement) is checked. If the loop is blocked, the controller stops the closing sequence or re-opens the gate immediately.

## Try This! (Challenges)
1. **Passage Tone**: Sound a buzzer on GP14 for 200 ms when the gate starts to open.
2. **Warning Beacon**: Add a red warning LED on GP15 that flashes rapidly while the gate is active or open.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate opens and closes constantly | Sensor reflection issue | IR obstacle sensors can be triggered by reflections from nearby walls or bright lights. Adjust the sensitivity potentiometer on the sensor board. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [37 - Pico IR Obstacle](../../beginner/37-pico-ir-obstacle.md)
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [58 - Pico LCD Print](58-pico-lcd-print.md)
