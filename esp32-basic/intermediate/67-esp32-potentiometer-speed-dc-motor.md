# 67 - ESP32 Potentiometer Speed DC Motor

Control the speed of a DC motor using a potentiometer and display the current speed percentage and direction on a 16x2 I2C LCD.

## Goal
Learn how to use analog input to adjust PWM outputs dynamically through the LEDC controller while outputting status reports to a character LCD display.

## What You Will Build
A potentiometer reads values on GPIO 34. The lower half of the potentiometer controls speed in reverse, the upper half controls speed forward, and the middle serves as a "dead-zone" stop. The 16x2 I2C LCD shows the speed percentage and direction.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor (3–12 V) | `dc_motor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Left leg | 3V3 | Red | Power rail |
| Potentiometer | Right leg | GND | Black | Ground rail |
| Potentiometer | Wiper | GPIO34 | Yellow | ADC Speed Input |
| L298N Module | IN1 | GPIO18 | Yellow | Direction Control 1 |
| L298N Module | IN2 | GPIO19 | Green | Direction Control 2 |
| L298N Module | ENA | GPIO5 | Orange | PWM Speed Control |
| L298N Module | GND | GND | Black | Common Ground |
| L298N Module | VCC | External 5V/Vin | Red | Motor Power |
| I2C LCD | VCC | 5V (Vin) | Red | LCD Power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Remove the ENA jumper from the L298N motor driver module and connect the ENA pin to GPIO 5. Make sure the external motor power supply GND is tied to the ESP32 GND.

## Code
```cpp
// Potentiometer Speed DC Motor with LCD Display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int POT_PIN = 34;
const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;

const int PWM_CHAN = 0;
const int PWM_FREQ = 1000;
const int PWM_RES = 8; // 8-bit: 0-255

LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  
  // Set up PWM
  ledcSetup(PWM_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, PWM_CHAN);
  
  // Initialise LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Motor Controller");
  
  delay(1000);
  lcd.clear();
}

void loop() {
  int raw = analogRead(POT_PIN); // 0 to 4095
  
  int speed = 0;
  String dir = "STOP";
  
  // Dead zone check (middle position 1900 - 2200)
  if (raw >= 1900 && raw <= 2200) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    ledcWrite(PWM_CHAN, 0);
    speed = 0;
    dir = "STOP";
  } 
  // Reverse (0 - 1899)
  else if (raw < 1900) {
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    // Map raw value to PWM duty cycle (0 is full speed, 1899 is slow speed)
    int duty = map(raw, 1899, 0, 0, 255);
    ledcWrite(PWM_CHAN, duty);
    speed = map(duty, 0, 255, 0, 100);
    dir = "REV";
  } 
  // Forward (2201 - 4095)
  else {
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    // Map raw value to PWM duty cycle (2201 is slow speed, 4095 is full speed)
    int duty = map(raw, 2201, 4095, 0, 255);
    ledcWrite(PWM_CHAN, duty);
    speed = map(duty, 0, 255, 0, 100);
    dir = "FWD";
  }

  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Dir: ");
  lcd.print(dir);
  lcd.print("      ");
  
  lcd.setCursor(0, 1);
  lcd.print("Speed: ");
  lcd.print(speed);
  lcd.print("%   ");
  
  Serial.print("Raw: "); Serial.print(raw);
  Serial.print(" | Dir: "); Serial.print(dir);
  Serial.print(" | Speed: "); Serial.print(speed); Serial.println("%");
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, **L298N**, **DC Motor**, and **16x2 I2C LCD** onto the canvas.
2. Connect Potentiometer wiper to **GPIO34**, IN1/IN2 to **GPIO18/GPIO19**, ENA to **GPIO5**, and the LCD SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer and observe the motor direction and speed update on the canvas and LCD.

## Expected Output
Serial Monitor:
```
Raw: 2048 | Dir: STOP | Speed: 0%
Raw: 4095 | Dir: FWD | Speed: 100%
Raw: 0 | Dir: REV | Speed: 100%
```

LCD Display (at full forward speed):
```
Dir: FWD
Speed: 100%
```

## Expected Canvas Behavior
* The motor is stopped when the potentiometer slider is centered.
* Sliding the potentiometer right causes the motor to spin clockwise at increasing speeds.
* Sliding the potentiometer left causes the motor to spin counter-clockwise at increasing speeds.
* The LCD displays the direction (FWD/REV/STOP) and calculated speed percentage.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `raw >= 1900 && raw <= 2200` | Creates a dead-zone buffer so the motor stays stopped at the potentiometer's center. |
| `map(raw, 1899, 0, 0, 255)` | Maps the reverse range so lower ADC values produce higher PWM speeds. |
| `map(raw, 2201, 4095, 0, 255)` | Maps the forward range so higher ADC values produce higher PWM speeds. |
| `ledcWrite(PWM_CHAN, duty)` | Applies the mapped duty cycle to GPIO 5. |

## Hardware & Safety Concept: Inductive Isolation and Dead-zone Protection
In motor control circuits, it is important to isolate the high-current noise of the motor coils from the microchip's sensitive logic. The L298N accomplishes this using optical couplers or driver logic gates. A dead-zone prevents rapid oscillations or motor twitching near the threshold where direction changes, protecting the motor gears and H-bridge transistors from sudden current reversals.

## Try This! (Challenges)
1. **Dynamic Brake**: Implement active motor braking when in the STOP dead zone by driving both IN1 and IN2 HIGH.
2. **Acceleration Ramp**: Smooth out changes in speed so the motor doesn't accelerate instantly.
3. **Over-Current Trip**: Set a limit parameter in code that turns off the motor if it runs at 100% speed for more than 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor humming but not spinning | Duty cycle too low to overcome friction | Change minimum mapped speed offset to start higher (e.g. 50 instead of 0) |
| Motor moves in opposite direction | Motor terminals reversed | Swap OUT1 and OUT2 wires at the L298N terminals |
| LCD display lags | Too many long delay functions | Ensure `delay()` in the loop is kept small (100ms or less) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - ESP32 DC Motor Speed Scaling](54-esp32-dc-motor-speed-scaling.md)
- [66 - ESP32 Servo Position Display](66-esp32-servo-position-display.md)
- [68 - ESP32 Stepper Motor Position Log](68-esp32-stepper-motor-position-log.md)
