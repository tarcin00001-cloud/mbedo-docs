# 145 - Pico Servo Radar OLED

Build a polar scanning radar console that displays target positions and sweeps on an OLED, and sounds pitch-varying chimes based on obstacle distance.

## Goal
Learn how to sweep a servo motor, measure target distances, draw a graphical circular scan grid on SSD1306 displays, and map distances to pitch-varying alert tones on a passive buzzer.

## What You Will Build
An advanced polar sonar radar system:
- **Servo Motor (GP10)**: Sweeps an ultrasonic sensor from 0° to 180° in 10° steps.
- **HC-SR04 Rangefinder (Trig GP14, Echo GP15)**: Scans for obstacles.
- **Passive Buzzer (GP11)**: Beeps with a pitch proportional to proximity (higher pitch = closer target).
- **SSD1306 OLED (GP4, GP5)**: Draws a circular radar scan map, the active scanning vector, target dots, and logs coordinates at the bottom (e.g. `A:120 D:45`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Servo Motor | Signal | GP10 | Radar sweep motor |
| HC-SR04 | Trig / Echo | GP14 / GP15 | Sensor lines |
| Passive Buzzer | VCC (+) | GP11 | Sound pitch alert |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Servo.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int SERVO_PIN  = 10;
const int BUZZER_PIN = 11;
const int TRIG_PIN   = 14;
const int ECHO_PIN   = 15;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

Servo radarServo;

int angle = 0;
int sweepDir = 10; // Sweep step size

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  radarServo.attach(SERVO_PIN);
  radarServo.write(angle);

  pinMode(BUZZER_PIN, OUTPUT);
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

  // Trigger ultrasonic measurement
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  if (distance < 0 || distance > 100) {
    distance = 100; // Limit scan range to 100 cm
  }

  // Sound warning pitch based on distance (closer target = higher pitch)
  if (distance < 50) {
    int frequency = 1000 + (50 - distance) * 20; // 1000 Hz to 2000 Hz
    tone(BUZZER_PIN, frequency, 50);
  } else {
    noTone(BUZZER_PIN);
  }

  // Draw radar HUD on OLED
  display.clearDisplay();

  // Draw circular grid frames
  display.drawCircle(64, 50, 45, SSD1306_WHITE);
  display.drawCircle(64, 50, 25, SSD1306_WHITE);
  display.drawFastHLine(19, 50, 90, SSD1306_WHITE); // Ground grid line

  // Calculate sweep vector coordinates using linear approximations
  int xOffset = (angle - 90) * 45 / 90;
  int targetX = 64 + xOffset;
  
  int absOffset = angle - 90;
  if (absOffset < 0) { absOffset = -absOffset; }
  int targetY = 50 - (45 - (absOffset * 25 / 90));

  // Draw active scanning vector line
  display.drawLine(64, 50, targetX, targetY, SSD1306_WHITE);

  // Draw target dot if obstacle is detected closer than 50 cm
  if (distance < 50) {
    int targetDistX = 64 + (xOffset * distance / 100);
    int targetDistY = 50 - ((50 - (absOffset * 25 / 90)) * distance / 100);
    display.fillCircle(targetDistX, targetDistY, 3, SSD1306_WHITE);
  }

  // Print text coordinates at the bottom (Row 3)
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 54);
  display.print("Ang:");
  display.print(angle);
  display.print("  Dist:");
  display.print(distance);
  display.print("cm");

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
1. Drag the **Raspberry Pi Pico**, **Servo**, **HC-SR04 Sensor**, **SSD1306 OLED**, and **Passive Buzzer** onto the canvas.
2. Connect Servo to **GP10**, Buzzer to **GP11**, HC-SR04 to **GP14/GP15**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the target obstacle distance on canvas and watch the OLED radar map and listen to the pitch-varying chime.

## Expected Output

Terminal:
```
Simulation active. Advanced OLED polar sonar radar engine online.
```

## Expected Canvas Behavior
* Radar Screen: Displays target grids. Scanning vector line sweeps back and forth.
* Coordinates: Bottom row logs `Ang: 90  Dist: 45cm`.
* Buzzer: plays pitch-varying warning chimes if a target enters the 50 cm alert zone.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `1000 + (50 - distance) * 20` | Dynamic frequency mapper, increasing buzzer pitch as the target gets closer. |

## Hardware & Safety Concept: Acoustic Warning Interfaces
In sonar systems, combining visual displays with auditory feedback is standard practice. Pitch-varying warning sounds (e.g. increasing frequency as targets get closer) warn operators of incoming hazards without requiring them to look at the screen constantly, improving response times.

## Try This! (Challenges)
1. **Critical Stop Indicator**: Light a warning LED on GP13 if the distance drops below 15 cm.
2. **Scan speed scaling**: Adjust the sweep step size based on distance (e.g. scan slower when targets are detected to improve accuracy).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click but no pitch change | Active buzzer used | Confirm you are using a **passive** buzzer. Active buzzers cannot modify output frequencies dynamically. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [115 - Pico Servo Radar](../intermediate/115-pico-servo-radar.md)
- [118 - Pico Ultrasonic OLED](../intermediate/118-pico-ultrasonic-oled.md)
- [128 - Pico Radar OLED](128-pico-radar-oled.md)
