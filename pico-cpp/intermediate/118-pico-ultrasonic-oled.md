# 118 - Pico Ultrasonic OLED

Build an digital rangefinder console that displays target distances and safety zones on an OLED screen.

## Goal
Learn how to read ultrasonic sensors, map distance values, and display graphic range metrics on SSD1306 OLED screens.

## What You Will Build
A digital tape measure HUD:
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Measures distance.
- **SSD1306 OLED (GP4, GP5)**: Displays the calculated distance in centimeters and draws a bar graph showing how close objects are.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Resistors | `resistor` | Optional | Yes (voltage divider) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | VCC | 5V | Sensor power |
| HC-SR04 | Trig | GP14 | Trigger output |
| HC-SR04 | Echo | GP15 | Echo input (requires 5V to 3.3V divider) |
| HC-SR04 | GND | GND | Ground reference |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int TRIG_PIN = 14;
const int ECHO_PIN = 15;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  digitalWrite(TRIG_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  // Generate trigger pulse
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  // Limit range readings
  if (distance < 0 || distance > 400) {
    distance = 400;
  }

  // Map distance (0-400cm) to screen bar width (0-128 pixels)
  // Inverting so the bar fills up as objects get closer
  int proximityWidth = 128 - (distance * 128 / 400);

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("RANGE FINDER");

  // Display distance digits
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 20);
  display.print("Distance: ");
  display.print(distance);
  display.print(" cm");

  // Zone status message
  display.setCursor(10, 32);
  if (distance < 30) {
    display.print("ZONE: !!! CLOSE !!!");
  } else if (distance >= 30 && distance < 100) {
    display.print("ZONE: WARNING");
  } else {
    display.print("ZONE: SAFE");
  }

  // Draw proximity bar graph border
  display.drawRect(0, 46, 128, 14, SSD1306_WHITE);

  // Fill proximity bar proportionally
  if (proximityWidth > 0) {
    display.fillRect(2, 48, proximityWidth - 4, 10, SSD1306_WHITE);
  }

  display.display();

  delay(200); // 5 updates per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect HC-SR04: **Trig** to **GP14**, **Echo** to **GP15** (with divider). Connect OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the HC-SR04 distance slider and watch the OLED gauge fill up as the target gets closer.

## Expected Output

Terminal:
```
Simulation active. OLED proximity rangefinder online.
```

## Expected Canvas Behavior
| Target Distance | OLED Status Readout | OLED Proximity Bar |
| --- | --- | --- |
| Far (> 200 cm) | `ZONE: SAFE` | Narrow bar |
| Medium (60 cm) | `ZONE: WARNING` | Half filled |
| Close (< 25 cm) | `ZONE: !!! CLOSE !!!` | **Almost fully filled** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `128 - (distance * 128 / 400)` | Inverts the scale mapping so the bar fills up as the object gets closer, creating a collision proximity warning bar. |

## Hardware & Safety Concept: Collision Proximity Bars
In automotive parking sensors, visual proximity bars are inverted. A full bar indicates the target is dangerously close (requiring immediate braking), while an empty bar shows clear space. This layout is standard in driver warning systems, helping drivers estimate distance without looking at numeric values.

## Try This! (Challenges)
1. **Audio Alarm**: Connect a buzzer on GP10 and sound a warning beep that plays faster as the distance drops.
2. **Reverse Display**: Modify the code to display distance in inches instead of centimeters.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Distance display stays stuck at maximum | Echo pin wire disconnected | Check the voltage divider wiring. If GP15 does not receive the return pulse, `pulseIn` returns 0 (which the code bounds to 400 cm). |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [75 - Pico Ultrasonic Serial](../intermediate/75-pico-ultrasonic-serial.md)
- [77 - Pico Ultrasonic LCD](../intermediate/77-pico-ultrasonic-lcd.md)
- [115 - Pico Servo Radar](115-pico-servo-radar.md)
