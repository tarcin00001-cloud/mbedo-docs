# 77 - HC-SR04 Distance Bar Graph LCD

Display distance measurements from an HC-SR04 sensor on a 16x2 I2C LCD, complete with a visual horizontal bar graph representation.

## Goal
Learn how to scale sensor range inputs to fit an LCD character line, creating a visual bar indicator representing proximity without using loops or array buffers.

## What You Will Build
An HC-SR04 sensor reads distance (Trig on GPIO 14, Echo on GPIO 15). A 16x2 I2C LCD displays the distance in centimeters on Row 1 and prints a graphical bar on Row 2 — the closer the object, the longer the bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | VCC | 5V | Red | Power input |
| HC-SR04 Sensor | Trig | GPIO 14 | Orange | Trigger control line |
| HC-SR04 Sensor | Echo | GPIO 15 | Yellow | Echo signal line |
| HC-SR04 Sensor | GND | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The SDA and SCL pins on the LCD connect to the hardware I2C0 interface on the ARIES board: SDA0 (GP17) and SCL0 (GP16). Verify that connections are solid.

## Code
```cpp
// HC-SR04 Distance Bar Graph LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Mapping limits for bar graph (in cm)
const int MIN_DIST = 5;   // Maximum bar length (full scale close)
const int MAX_DIST = 80;  // Minimum bar length (empty scale far)

void setup() {
  Wire.begin();
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  digitalWrite(TRIG_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Range Finder");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  delay(1000);
  lcd.clear();
}

void loop() {
  // Trigger sensor pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  long duration = pulseIn(ECHO_PIN, HIGH);
  float distance = (duration * 0.0343) / 2.0;
  
  lcd.setCursor(0, 0);
  if (duration == 0 || distance > 400.0) {
    lcd.print("Out of Range    ");
    lcd.setCursor(0, 1);
    lcd.print("                "); // Clear bar graph row
  } else {
    lcd.print("Dist: ");
    lcd.print(distance, 1);
    lcd.print(" cm    "); // Trailing spaces
    
    // Calculate bar graph columns (invert mapping: closer = longer bar)
    int numBlocks = map((int)distance, MAX_DIST, MIN_DIST, 0, 16);
    if (numBlocks < 0) numBlocks = 0;
    if (numBlocks > 16) numBlocks = 16;
    
    lcd.setCursor(0, 1);
    // Draw bar graph using switch-case to remain 100% loop-free
    switch (numBlocks) {
      case 0:  lcd.print("                "); break;
      case 1:  lcd.print("=               "); break;
      case 2:  lcd.print("==              "); break;
      case 3:  lcd.print("===             "); break;
      case 4:  lcd.print("====            "); break;
      case 5:  lcd.print("=====           "); break;
      case 6:  lcd.print("======          "); break;
      case 7:  lcd.print("=======         "); break;
      case 8:  lcd.print("========        "); break;
      case 9:  lcd.print("=========       "); break;
      case 10: lcd.print("==========      "); break;
      case 11: lcd.print("===========     "); break;
      case 12: lcd.print("============    "); break;
      case 13: lcd.print("=============   "); break;
      case 14: lcd.print("==============  "); break;
      case 15: lcd.print("=============== "); break;
      case 16: lcd.print("================"); break;
    }
  }
  
  delay(300); // refresh rate
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-SR04 Sensor**, and **I2C LCD Display** components onto the canvas.
2. Wire the sensor: **Trig** to **GPIO 14**, **Echo** to **GPIO 15**, **VCC** to **5V**, and **GND** to **GND**.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Change the distance slider on the HC-SR04 widget and watch the bar graph on the LCD screen scale dynamically.

## Expected Output
LCD Display (at 20 cm distance):
```
Dist: 20.0 cm
============
```

## Expected Canvas Behavior
* Row 1 displays the current distance reading.
* Row 2 displays a horizontal bar of "=" characters. As the distance slider moves closer to 5 cm, the number of "=" blocks grows. As it moves toward 80 cm, the bar shrinks.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `map(distance, MAX_DIST, MIN_DIST, 0, 16)` | Maps the range inversely so smaller distances produce longer bars. |
| `switch (numBlocks)` | Updates the line text in one transaction to avoid using `for` loops. |
| `lcd.print("= ")` | Custom character block outputs for visual bar construction. |

## Hardware & Safety Concept
* **Sensory HUD Layouts**: Graphical bar representations allow operators to quickly assess proximity status without needing to read precise numbers. They are widely used in industrial level indicators, audio volume monitors, and vehicle dashboard displays.

## Try This! (Challenges)
1. **Reverse logic**: Change the mapping so the bar grows as the distance increases (useful for liquid tank levels).
2. **Beep Alarm integration**: Add an active buzzer on GPIO 12 that beeps at an interval proportional to the bar graph length.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph flickers | LCD cleared too often | Avoid calling `lcd.clear()` inside the loop; overwrite characters with spaces instead. |
| Bar graph doesn't change | Constrain bounds too tight | Check `MIN_DIST` and `MAX_DIST` bounds in the code. |
| LCD output is garbled | Shared I2C address conflict | Verify address configurations of all devices connected to the I2C bus. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - HC-SR04 Proximity Sensor Serial Logs](75-hc-sr04-proximity-sensor-serial.md)
- [76 - HC-SR04 Proximity Alarm](76-hc-sr04-proximity-alarm.md)
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
