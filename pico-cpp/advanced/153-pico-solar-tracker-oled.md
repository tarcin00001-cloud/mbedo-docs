# 153 - Pico Solar Tracker OLED

Build an automated solar tracker that rotates a servo platform to track light and displays real-time LDR coordinates on an OLED screen.

## Goal
Learn how to read dual analog sensors (LDRs), drive servo motors, and display tracking metrics and differences on SSD1306 OLED screens.

## What You Will Build
An adjustable solar tracking model:
- **Left LDR Photoresistor (GP26)**: Measures light on the left side.
- **Right LDR Photoresistor (GP27)**: Measures light on the right side.
- **Servo Motor (GP10)**: Steers the solar panel platform left or right to balance the light levels.
- **SSD1306 OLED (GP4, GP5)**: Displays Left/Right light intensities, calculated differences, and active steering status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Servo Motor | `servo` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage dividers) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Left LDR | Signal (Pin 2) | GP26 | Analog input (requires 10k pull-down) |
| Right LDR | Signal (Pin 2) | GP27 | Analog input (requires 10k pull-down) |
| Servo Motor | Signal | GP10 | Rotating platform control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int LDR_LEFT  = 26;
const int LDR_RIGHT = 27;
const int SERVO_PIN = 10;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

Servo trackerServo;

int servoAngle = 90; // Start at center position
const int tolerance = 100; // Analog noise tolerance band

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  trackerServo.attach(SERVO_PIN);
  trackerServo.write(servoAngle);
  
  pinMode(LDR_LEFT, INPUT);
  pinMode(LDR_RIGHT, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
  delay(1000);
}

void loop() {
  int valLeft  = analogRead(LDR_LEFT);
  int valRight = analogRead(LDR_RIGHT);

  // Calculate the difference
  int diff = valLeft - valRight;
  int absDiff = diff;
  if (absDiff < 0) { absDiff = -absDiff; }

  char steerState[10] = "LOCKED";

  // Adjust servo position if difference exceeds tolerance
  if (absDiff > tolerance) {
    if (diff > 0) {
      servoAngle = servoAngle + 3;
      if (servoAngle > 180) { servoAngle = 180; }
      steerState[0] = 'L'; steerState[1] = 'E'; steerState[2] = 'F'; steerState[3] = 'T'; steerState[4] = '\0';
    } else {
      servoAngle = servoAngle - 3;
      if (servoAngle < 0) { servoAngle = 0; }
      steerState[0] = 'R'; steerState[1] = 'I'; steerState[2] = 'G'; steerState[3] = 'H'; steerState[4] = 'T'; steerState[5] = '\0';
    }
    trackerServo.write(servoAngle);
  }

  // Draw HUD on OLED
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("SOLAR TRACKER");

  // Display LDR values
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  
  // Row 1: Left & Right readings
  display.setCursor(8, 20);
  display.print("L: ");
  display.print(valLeft);
  display.print(" | R: ");
  display.print(valRight);

  // Row 2: Active Angle
  display.setCursor(8, 34);
  display.print("Angle: ");
  display.print(servoAngle);
  display.print(" deg");

  // Row 3: Action status
  display.setCursor(8, 48);
  display.print("Steer: ");
  display.print(steerState);

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(50); // Control tracking speed
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two LDRs**, **Servo Motor**, and **SSD1306 OLED** onto the canvas.
2. Connect Left LDR to **GP26**, Right LDR to **GP27**, Servo to **GP10**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the light level of either LDR up and watch the servo motor rotate and the OLED update.

## Expected Output

Terminal:
```
Simulation active. Solar tracker OLED HUD active.
```

## Expected Canvas Behavior
* Equal light: OLED reads `Steer: LOCKED`, Servo at 90°.
* Slide Left LDR up: OLED reads `Steer: LEFT`, Servo rotates left toward 180°.
* Slide Right LDR up: OLED reads `Steer: RIGHT`, Servo rotates right toward 0°.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `absDiff > tolerance` | Prevents the servo from oscillating back and forth continuously (jittering) due to minor analog noise. |

## Hardware & Safety Concept: Solar Tracker Efficiency
Solar panels generate maximum electrical power when they are perpendicular to the incoming sun rays. Single-axis solar trackers rotate the panel throughout the day to follow the sun's path from East to West, increasing power generation efficiency by up to 25% compared to fixed-angle panels.

## Try This! (Challenges)
1. **Indicator LEDs**: Add a Red LED on GP15 that turns ON when the platform is rotating, and a Green LED on GP14 when tracking is locked.
2. **Dynamic Step Size**: Modify the code to adjust the servo step speed dynamically based on the size of the light difference (e.g. steering faster if the light difference is large).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo rotates away from the light | Sensor channels swapped | Swap the GP26 and GP27 input pin definitions in your code, or swap the physical LDR positions. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [34 - Pico LDR Sensor](../../beginner/34-pico-ldr-sensor.md)
- [46 - Pico Pot Servo](../../beginner/46-pico-pot-servo.md)
- [134 - Pico Solar Tracker](134-pico-solar-tracker.md)
