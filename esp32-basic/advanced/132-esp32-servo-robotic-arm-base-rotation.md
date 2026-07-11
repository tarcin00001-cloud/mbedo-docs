# 132 - ESP32 Servo Robotic Arm Base Rotation

Build a precise manual joint controller for a robotic arm, mapping a potentiometer analog input through a noise-filtering average buffer to steer a base sweep servo, with angle feedback on an I2C LCD.

## Goal
Learn how to apply running average filters to smooth out analog noise, map ADC readings to servo degrees, and display target joint coordinates.

## What You Will Build
A potentiometer is connected to GPIO 34. A SG90 servo motor (base joint) is connected to GPIO 13. A 16x2 I2C LCD displays the joint angles. The code continuously reads the potentiometer, passes the value through a running average filter to eliminate jitter, maps it to a 0°–180° range, and writes the position to the servo.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Joint angle analog input |
| Potentiometer | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Servo Motor | Signal | GPIO13 | Orange | PWM joint control |
| Servo Motor | VCC / GND | 5V / GND | Red / Black | Power rails |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |

> **Wiring tip:** Ground pins of the potentiometer, servo, and LCD must all be connected to the ESP32 GND pin to establish a shared reference.

## Code
```cpp
// Servo Robotic Arm Base Rotation
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int POT_PIN = 34;
const int SERVO_PIN = 13;

Servo baseServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Filtering Parameters
const int WINDOW_SIZE = 10;
int readBuffer[WINDOW_SIZE];
int readIndex = 0;
long totalSum = 0;

int filterAnalog(int newValue) {
  // Subtract last reading
  totalSum = totalSum - readBuffer[readIndex];
  // Write new reading
  readBuffer[readIndex] = newValue;
  // Add new reading to sum
  totalSum = totalSum + newValue;
  // Advance index
  readIndex = (readIndex + 1) % WINDOW_SIZE;
  // Return average
  return totalSum / WINDOW_SIZE;
}

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Robotic Arm Base");
  lcd.setCursor(0, 1);
  lcd.print("Initialising...");
  
  // Initialise filter buffer with startup read
  int initialRead = analogRead(POT_PIN);
  for (int i = 0; i < WINDOW_SIZE; i++) {
    readBuffer[i] = initialRead;
  }
  totalSum = (long)initialRead * WINDOW_SIZE;
  
  baseServo.attach(SERVO_PIN);
  baseServo.write(90); // Start centered
  
  delay(1000);
  lcd.clear();
}

void loop() {
  int rawValue = analogRead(POT_PIN);
  
  // Apply running average filter to eliminate signal noise
  int smoothedValue = filterAnalog(rawValue);
  
  // Map 12-bit ADC (0-4095) to servo degrees (0-180)
  int angle = map(smoothedValue, 0, 4095, 0, 180);
  
  // Command servo
  baseServo.write(angle);
  
  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Joint: BASE      ");
  lcd.setCursor(0, 1);
  lcd.print("Angle: ");
  lcd.print(angle);
  lcd.print((char)223); // Degree symbol
  lcd.print("   ");    // Clear trailing digits
  
  Serial.print("Raw: "); Serial.print(rawValue);
  Serial.print(" | Smoothed: "); Serial.print(smoothedValue);
  Serial.print(" | Angle: "); Serial.println(angle);
  
  delay(50); // 20Hz update rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, **Servo**, and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Servo to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget on the canvas. Watch the servo sweep and LCD angle change.

## Expected Output
Serial Monitor:
```
Raw: 2045 | Smoothed: 2048 | Angle: 90
Raw: 4095 | Smoothed: 4095 | Angle: 180
Raw: 0 | Smoothed: 0 | Angle: 0
```

LCD Display:
```
Joint: BASE
Angle: 90°
```

## Expected Canvas Behavior
* Sliding the potentiometer completely left moves the servo widget arm to 0° and updates the LCD to "Angle: 0°".
* Sliding the potentiometer completely right moves the servo to 180° and updates the LCD to "Angle: 180°".
* The filter removes jitter, resulting in a smooth sweep on the canvas.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `filterAnalog(rawValue)` | Implements a **rolling average window** to filter out electromagnetic ADC noise. |
| `map(..., 0, 4095, 0, 180)` | Scales the 12-bit analog input range to match the servo's physical range. |
| `baseServo.write(angle)` | Outputs the corresponding PWM duty cycle to position the servo. |

## Hardware & Safety Concept: Analog Filtering and Servo Jitter
High-current devices like motors draw power unevenly, creating voltage fluctuations on the ESP32's power rails. This fluctuation causes noise on the analog pins, making the ADC readings fluctuate rapidly. This causes **servo jitter** (where the servo constantly vibrates and draws excess current, causing it to wear out). Implementing a **running average filter** in code smooths out these fluctuations, protecting the servo from jitter.

## Try This! (Challenges)
1. **Dynamic Filter Window**: Increase `WINDOW_SIZE` to 20. Observe how the servo becomes smoother but reacts with more latency.
2. **Preset Position button**: Add a button on GPIO 4 that moves the servo to 90° immediately, overriding the potentiometer.
3. **Safety Limit bounds**: Constrain the servo movement between 30° and 150° in code to prevent hitting mechanical arm stops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo vibrates constantly | Analog noise or weak power supply | Connect the servo VCC to an external 5V source; increase `WINDOW_SIZE` |
| LCD does not print data | Contrast screw setting | Adjust the contrast potentiometer on the back of the LCD module |
| Servo moves in opposite direction of knob | Potentiometer wired backward | Swap the VCC and GND wires of the potentiometer |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Potentiometer Servo Control](../beginner/46-esp32-potentiometer-servo-control.md)
- [133 - ESP32 2-Axis Robotic Arm Joint Controller](133-esp32-2-axis-robotic-arm-joint-controller.md)
- [66 - ESP32 Servo Position Display](66-esp32-servo-position-display.md)
