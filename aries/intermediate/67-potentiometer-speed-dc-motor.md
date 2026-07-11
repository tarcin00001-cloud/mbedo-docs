# 67 - Potentiometer Speed DC Motor (with LCD)

Control the speed of a DC motor using an analog potentiometer and display the live speed percentage on a 16x2 I2C LCD screen using the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs from the ADC, map values to PWM signals, control motor speed via an L298N driver, and update an I2C display dynamically without using loops inside C++ code.

## What You Will Build
An interactive motor control console where rotating the potentiometer dial adjusts the DC motor's rotation speed and displays the current speed percentage (0% to 100%) on the LCD screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 5V DC Motor | `motor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3V3 | Red | Analog voltage supply |
| Potentiometer | Pin 2 (Wiper) | ADC0 (GP26) | White | Analog output signal |
| Potentiometer | Pin 3 (GND) | GND | Black | Ground reference |
| L298N Driver | IN1 | GPIO 14 | Blue | Control input 1 |
| L298N Driver | IN2 | GPIO 15 | Yellow | Control input 2 |
| L298N Driver | ENA | GPIO 13 | Orange | PWM speed control line |
| L298N Driver | VCC | 5V | Red | Driver board power |
| L298N Driver | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| DC Motor | Terminal 1 | OUT1 | Red | Driver output 1 |
| DC Motor | Terminal 2 | OUT2 | Black | Driver output 2 |

> **Wiring tip:** Share the VCC and GND connections using a breadboard. Remove the ENA jumper on the L298N driver and connect ENA to GPIO 13.

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int POT_PIN = 26; // ADC0 is GP26
const int IN1 = 14;
const int IN2 = 15;
const int ENA = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);
int lastPercent = -1;

void setup() {
  Wire.begin();

  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);

  // Set motor rotation direction to forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Motor Control");
}

void loop() {
  int potValue = analogRead(POT_PIN);

  // Map 12-bit analog input (0 to 4095) to PWM duty cycle (0 to 255)
  int pwmValue = potValue * 255 / 4095;

  // Map to speed percentage (0 to 100)
  int percent = potValue * 100 / 4095;

  analogWrite(ENA, pwmValue); // Write speed PWM output to ENA

  // Only update screen if speed percentage changes to prevent flickering
  if (percent != lastPercent) {
    lcd.setCursor(0, 1);
    lcd.print("Speed: ");
    lcd.print(percent);
    lcd.print("%   "); // Trailing spaces overwrite old digits
    lastPercent = percent;
  }

  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Potentiometer**, **L298N Driver**, **DC Motor**, and **I2C LCD Display** components onto the canvas.
2. Connect the Potentiometer: **Pin 1** to **3V3**, **Wiper** to **ADC0 (GP26)**, and **Pin 3** to **GND**.
3. Connect the L298N: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **ENA** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**. Wire the motor.
4. Connect the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
ADC: 2048, Speed: 50%
```

## Expected Canvas Behavior
* Adjusting the potentiometer dial in the simulator changes the DC motor rotation speed. The LCD displays `Motor Control` on the top row, and updating speed values (e.g. `Speed: 52%`) on the bottom row.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(POT_PIN)` | Reads potentiometer analog input (0 to 4095). |
| `potValue * 255 / 4095` | Calculates corresponding 8-bit PWM value (0 to 255). |
| `analogWrite(ENA, pwmValue)` | Pushes the PWM signal to the L298N speed enable pin. |
| `percent != lastPercent` | Only redraws the screen on speed changes to avoid flickering. |

## Hardware & Safety Concept
* **H-Bridge Voltage Drops**: L298N drivers are bipolar H-bridges, which have an internal voltage drop of about 1.4V to 2V across the power transistors. Running a 5V motor requires feeding at least 7V to the driver's VCC terminal if full 5V motor speed is desired.
* **Separate Ground Paths**: High-current motor currents can introduce digital noise back into the ground lines, causing the microcontroller's ADC reads to fluctuate. Always keep power grounds separate, joining them at a single common reference point (star grounding).

## Try This! (Challenges)
1. **Interactive Direction**: Add a button on GPIO 16 to toggle rotation direction. Update row 0 on the LCD to show either `DIR: FORWARD` or `DIR: REVERSE`.
2. **Speed Bar Graph**: Draw a simple progress bar on the LCD using standard characters (e.g. `>` or `=`) representing the speed level (0 to 10 blocks).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD updates slowly or flickers constantly | Missing state change check | Ensure the `percent != lastPercent` check is implemented to prevent printing to the LCD on every loop cycle. |
| Motor spins at 100% speed regardless of dial | ENA jumper installed | Verify the jumper on the L298N ENA pins is removed, and GPIO 13 connects to the driver's ENA pin. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - DC Motor Speed Scaling (PWM ENA)](54-dc-motor-speed-scaling.md)
- [66 - Servo Position Display](66-servo-position-display.md)
