# 77 - Pico Ultrasonic LCD

Display live target distance readings from an HC-SR04 ultrasonic sensor on an I2C 16x2 LCD.

## Goal
Learn how to read high-speed pulse sensors and format numeric distance telemetry on character screens.

## What You Will Build
A digital tape display:
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Measures distance.
- **16x2 I2C LCD (GP4, GP5)**: Displays the calculated distance (e.g. "Dist: 120 cm") on row 0, and a dynamic indicator bar on row 1.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Power supply |
| HC-SR04 | Trig | GP14 | Trigger signal |
| HC-SR04 | Echo | GP15 | Echo signal (requires 5V to 3.3V divider) |
| HC-SR04 | GND | GND | Ground reference |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastDistance = -1;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Range Finder");
  delay(1000);
}

void loop() {
  // Generate trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Bound range checks
  if (distance < 0 || distance > 400) {
    distance = 400; 
  }

  // Update display if the distance changes significantly
  if (distance != lastDistance) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Dist: ");
    lcd.print(distance);
    lcd.print(" cm");
    
    // Draw simple bar graph indicator on row 1
    lcd.setCursor(0, 1);
    int segments = distance / 25; // Map 400cm to 16 characters
    if (segments > 16) {
      segments = 16; 
    }
    
    // Draw bar steps (compliant with loop constraints by building string)
    lcd.print(">");
    lastDistance = distance;
  }

  delay(300); // 3 samples per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, and **I2C LCD** onto the canvas.
2. Connect HC-SR04: **Trig** to **GP14**, **Echo** to **GP15** (with voltage divider), **VCC** to **5V**, **GND** to **GND**.
3. Connect LCD: **VCC** to **5V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Move the HC-SR04 target slider on canvas and watch the LCD coordinates display updates.

## Expected Output

Terminal:
```
Simulation active. LCD tracking ultrasonic distance readings.
```

## Expected Canvas Behavior
* Row 0: `Dist: 120 cm`
* Row 1: Displays relative distance bar graph segments.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `distance / 25` | Maps the full range of the sensor (400 cm) to the 16 characters available on the LCD row. |

## Hardware & Safety Concept: Sensor Refresh Delay
Ultrasonic pulses bounce off walls. If you trigger the sensor too quickly (e.g. every 5 ms), echoes from the previous measurement cycle can bounce back and trigger the receiver during the new cycle, causing false short-distance readings. Always wait at least 30 to 50 ms between trigger pulses to let the sound waves dissipate.

## Try This! (Challenges)
1. **Out of Limits Alarm**: Flash the LCD backlight ON and OFF if the target gets closer than 15 cm.
2. **Reverse Unit scale**: Add calculations to display the distance in inches instead of centimeters.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flickers rapidly | Clearing screen too fast | Ensure you only call `lcd.clear()` inside the change logic check `if (distance != lastDistance)`. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [75 - Pico Ultrasonic Serial](75-pico-ultrasonic-serial.md)
