# 128 - Pico Radar OLED

Build a mini radar system that sweeps an ultrasonic sensor on a servo motor and draws a dynamic radar map on an OLED screen.

## Goal
Learn how to sweep a servo motor, measure distances, and draw corresponding lines and dots on a graphic SSD1306 OLED screen.

## What You Will Build
A radar sweep visualizer:
- **Servo Motor (GP10)**: Sweeps back and forth incrementally.
- **HC-SR04 Sensor (Trig GP14, Echo GP15)**: Measures distance at each step.
- **SSD1306 OLED (GP4, GP5)**: Draws a scanning line and a dot representing detected targets on the radar map.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Servo Motor | Signal | GP10 | Radar sweep motor |
| HC-SR04 | Trig | GP14 | Distance trigger |
| HC-SR04 | Echo | GP15 | Distance echo (with divider) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int SERVO_PIN = 10;
const int TRIG_PIN  = 14;
const int ECHO_PIN  = 15;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

Servo radarServo;

int angle = 0;
int sweepDir = 10; // Sweep increment step size (degrees)

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  radarServo.attach(SERVO_PIN);
  radarServo.write(angle);

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  digitalWrite(TRIG_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
  delay(1000);
}

void loop() {
  // Move servo to the current angle step
  radarServo.write(angle);
  delay(150); // Settle delay

  // Measure distance
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  if (distance < 0 || distance > 100) {
    distance = 100; // Limit scan range to 100 cm
  }

  // Draw radar sweeps on OLED
  display.clearDisplay();

  // Draw semi-circular radar frame
  display.drawCircle(64, 60, 50, SSD1306_WHITE);
  display.drawCircle(64, 60, 30, SSD1306_WHITE);
  display.drawFastHLine(14, 60, 100, SSD1306_WHITE); // Ground line

  // Calculate terminal sweep line coordinate using approximations (no trigonometry in interpreted)
  // Maps 0-180 degrees to relative horizontal offsets
  int xOffset = (angle - 90) * 50 / 90;
  int targetX = 64 + xOffset;
  
  // y coordinate drops as we get further from horizontal line
  int absOffset = angle - 90;
  if (absOffset < 0) { absOffset = -absOffset; }
  int targetY = 60 - (50 - (absOffset * 30 / 90));

  // Draw active scanning vector line
  display.drawLine(64, 60, targetX, targetY, SSD1306_WHITE);

  // Draw target dot if object is close (< 60 cm)
  if (distance < 60) {
    int targetDistX = 64 + (xOffset * distance / 100);
    int targetDistY = 60 - ((60 - (absOffset * 30 / 90)) * distance / 100);
    display.fillCircle(targetDistX, targetDistY, 3, SSD1306_WHITE);
  }

  display.display();

  // Increment sweep
  angle = angle + sweepDir;

  // Reverse direction at limit boundaries
  if (angle >= 180) {
    angle = 180;
    sweepDir = -10;
  }
  if (angle <= 0) {
    angle = 0;
    sweepDir = 10;
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Servo**, **HC-SR04 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect Servo to **GP10**, HC-SR04 to **GP14/GP15**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Move the HC-SR04 target obstacle closer on canvas and watch the target dot appear on the OLED radar map.

## Expected Output

Terminal:
```
Simulation active. OLED radar graphic renderer online.
```

## Expected Canvas Behavior
* Radar Screen: Displays a semi-circular target grid. A sweeping scan line rotates back and forth.
* Object detection: If an obstacle is closer than 60 cm, a bright target dot appears along the scanning vector at the measured distance.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.drawLine(64, 60, targetX, targetY, ...)` | Draws the active radar scanning sweep line originating from the bottom-center (64, 60). |

## Hardware & Safety Concept: Graphical Coordinate Projections
Generating polar radar map graphics requires converting angle and distance values into Cartesian coordinates (X, Y). While compile-based microcontrollers use trigonometric math libraries (`cos(rad)`, `sin(rad)`), interpreted mode runtimes avoid expensive decimal calculations by using linear ratio approximations (e.g. mapping coordinates using offset scales) to render drawings quickly.

## Try This! (Challenges)
1. **Critical Overrun Siren**: Sound a buzzer on GP11 if a target appears closer than 20 cm at any scan angle.
2. **Scan speed modifier**: Increase the sweep step to 15° to make the scan sweep faster.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Radar lines are distorted | Integer division error | Ensure coordinate offsets remain within the 128x64 display boundaries to prevent pixel overflow wraps. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [75 - Pico Ultrasonic Serial](../intermediate/75-pico-ultrasonic-serial.md)
- [115 - Pico Servo Radar](../intermediate/115-pico-servo-radar.md)
- [118 - Pico Ultrasonic OLED](../intermediate/118-pico-ultrasonic-oled.md)
