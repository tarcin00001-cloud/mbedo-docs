# 67 - Pico Motor LCD

Control the speed of a DC motor using a potentiometer and display the power level on a 16x2 LCD screen.

## Goal
Learn how to map analog inputs to PWM motor controls while reporting system status metrics on an I2C text display.

## What You Will Build
An adjustable fan speed panel:
- **Potentiometer (GP26)**: Adjusts speed.
- **DC Motor (via L298N)**: Speed scales from 0% to 100% duty cycle.
- **16x2 I2C LCD**: Displays current motor speed percentage (e.g. "Speed: 50 %").
- **L298N Pins**: ENA (GP13), IN1 (GP14), IN2 (GP15).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| L298N | ENA | GP13 | Speed PWM control |
| L298N | IN1 | GP14 | Direction control 1 |
| L298N | IN2 | GP15 | Direction control 2 |
| L298N | GND | GND | Shared ground |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int POT_PIN = 26;
const int ENA     = 13;
const int IN1     = 14;
const int IN2     = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastPercent = -1;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(POT_PIN, INPUT);
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);

  // Set motor forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Fan Controller");
}

void loop() {
  int rawValue = analogRead(POT_PIN);
  int pwmValue = rawValue / 16;
  int percent = pwmValue * 100 / 255;

  analogWrite(ENA, pwmValue);

  if (percent != lastPercent) {
    lcd.setCursor(0, 1);
    lcd.print("Speed: ");
    lcd.print(percent);
    lcd.print(" %    ");
    lastPercent = percent;
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, **L298N**, **DC Motor**, and **I2C LCD** onto the canvas.
2. Connect Potentiometer to **GP26**, L298N to **GP13-15**, and LCD to **GP4/5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the potentiometer dial to control fan speed and monitor the percentage on LCD.

## Expected Output

Terminal:
```
Simulation active. LCD rendering motor speed percentage.
```

## Expected Canvas Behavior
| Potentiometer Slider | Motor Spin Speed | LCD Row 1 Print |
| --- | --- | --- |
| Far Left | Stopped | `Speed: 0 %` |
| Center | Medium speed | `Speed: 50 %` |
| Far Right | Full speed | `Speed: 100 %` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pwmValue * 100 / 255` | Computes percentage ratio representing motor speed from 0 to 100. |

## Hardware & Safety Concept: Overload Prevention
Running DC motors at full load can draw high currents which can trigger thermal safety shut-offs on H-bridges. Monitoring motor status and restricting max speed levels during starting phases prevents sudden current surges, protecting battery blocks and driver chips.

## Try This! (Challenges)
1. **Direction Switch**: Connect a toggle switch to GP16 and update the code to change motor spin direction while updating the LCD to show "DIR: FWD" or "DIR: REV".
2. **Dynamic Bar**: Draw a dynamic speed bar on the LCD.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor doesn't spin | ENA jumper still connected | Ensure you physically remove the black jumper block on L298N ENA pin before connecting the GP13 wire. |

## Mode Notes
This multi-device I2C motor control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [54 - Pico Motor Speed](54-pico-motor-speed.md)
- [66 - Pico Servo LCD](66-pico-servo-lcd.md)
