# 77 - ESP32 HC-SR04 Distance Bar Graph LCD

Display distance measurements from an HC-SR04 sensor on a 16x2 I2C LCD, complete with a visual horizontal bar graph.

## Goal
Learn how to scale sensor range inputs to fit an LCD character array, creating a visual indicator representing proximity.

## What You Will Build
An HC-SR04 sensor reads distance on GPIO 5 (Trig) and GPIO 18 (Echo). The 16x2 I2C LCD displays the distance in centimeters on Row 1 and prints a graphical bar on Row 2 — the closer the object, the longer the bar.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic_sensor` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | Trig | GPIO5 | Orange | Trigger output |
| HC-SR04 Sensor | Echo | GPIO18 | Yellow | Echo input |
| HC-SR04 Sensor | VCC / GND | 5V / GND | Red / Black | Power |
| I2C LCD | VCC | 5V (Vin) | Red | LCD power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** The Trig, Echo, SDA, and SCL lines are connected to different pins so they don't interfere. In physical builds, add a voltage divider to drop the 5V Echo pulse down to a safe 3.3V before feeding it into GPIO 18.

## Code
```cpp
// HC-SR04 Distance Bar Graph LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN = 5;
const int ECHO_PIN = 18;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Mapping limits for bar graph (in cm)
const float MIN_DIST = 5.0;   // Maximum bar length (full scale close)
const float MAX_DIST = 80.0;  // Minimum bar length (empty scale far)

void setup() {
  Serial.begin(115200);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  digitalWrite(TRIG_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Range Finder");
  lcd.setCursor(0, 1);
  lcd.print("Ready...");
  
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
    lcd.print("Distance: ");
    lcd.print(distance, 1);
    lcd.print("cm   "); // Trailing spaces
    
    // Calculate bar graph columns (invert mapping: closer = longer bar)
    int numBlocks = map((int)distance, (int)MAX_DIST, (int)MIN_DIST, 0, 16);
    numBlocks = constrain(numBlocks, 0, 16);
    
    lcd.setCursor(0, 1);
    for (int i = 0; i < 16; i++) {
      if (i < numBlocks) {
        lcd.print("="); // Use "=" for bar representation
      } else {
        lcd.print(" ");
      }
    }
  }
  
  Serial.print("Dist: "); Serial.println(distance);
  delay(250); // 4Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **HC-SR04 Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Wire Trig to **GPIO5**, Echo to **GPIO18**, and LCD SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the distance slider on the sensor widget and observe the bar graph length changes on the LCD.

## Expected Output
Serial Monitor:
```
Dist: 45.2
Dist: 20.0
Dist: 5.5
```

LCD Display (at 20 cm distance):
```
Distance: 20.0cm
============
```

## Expected Canvas Behavior
* The top row of the LCD displays "Distance: XX.Xcm".
* The bottom row displays a horizontal line of "=" characters.
* As the distance slider moves closer to 5 cm, the number of "=" blocks grows. As it moves toward 80 cm, the bar shrinks.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `map(distance, MAX_DIST, MIN_DIST, 0, 16)` | Maps the range inversely so smaller distances produce longer bars. |
| `constrain(numBlocks, 0, 16)` | Limits the bar size to the LCD width (16 columns). |
| `lcd.print(" ")` | Overwrites old blocks with blank spaces to clear the line without flickering. |

## Hardware & Safety Concept: Sensor Visualizer Layouts
Graphical bar representations allow operators to quickly assess proximity status without needing to read precise numbers. They are widely used in industrial level indicators, audio volume monitors, and vehicle dashboard displays.

## Try This! (Challenges)
1. **Custom Solid Blocks**: Use the custom character generator (Project 59) to define a solid block character to replace the "=" symbol for a cleaner bar graph look.
2. **Reverse logic**: Change the mapping so the bar grows as the distance increases (useful for liquid tank levels).
3. **Beep Alarm integration**: Add a buzzer on GPIO 15 that beeps at an interval proportional to the bar graph length.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph flickers | LCD cleared too often | Avoid calling `lcd.clear()` inside the loop; overwrite characters with spaces instead |
| Bar graph doesn't change | Constrain bounds too tight | Check `MIN_DIST` and `MAX_DIST` bounds in the code |
| Distance display is erratic | Intermittent Echo signals | Wrap the distance measurement in a simple moving average filter |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [75 - ESP32 HC-SR04 Proximity Sensor Serial Logs](75-esp32-hcsr04-proximity-sensor-serial-logs.md)
- [76 - ESP32 HC-SR04 Proximity Alarm](76-esp32-hcsr04-proximity-alarm.md)
- [59 - ESP32 16×2 I2C LCD Custom Characters](59-esp32-lcd-custom-characters.md)
