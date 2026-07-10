# 144 - Pico Obstacle Avoider OLED

Build an autonomous obstacle-avoiding mobile robot that displays real-time rangefinder measurements and steering alerts on an OLED screen.

## Goal
Learn how to parse ultrasonic sensor distance telemetry, display graphic navigation HUDs on OLED screens, and steer dual H-bridge DC motors.

## What You Will Build
An autonomous vehicle with a graphical telemetry console:
- **HC-SR04 Sensor (Trig GP16, Echo GP17)**: Measures front obstacle distance.
- **L298N & Dual DC Motors (GP10-15)**: Steers the robot around barriers.
- **SSD1306 OLED (GP4, GP5)**: Displays the current obstacle distance in centimeters, a graphic occupancy bar, and active steering messages (e.g. "FORWARD" or "REVERTING").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-SR04 | Trig / Echo | GP16 / GP17 | Rangefinder lines |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right motor speed |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left motor direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right motor direction |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int TRIG_PIN = 16;
const int ECHO_PIN = 17;

const int ENA = 10; const int ENB = 11;
const int IN1 = 12; const int IN2 = 13;
const int IN3 = 14; const int IN4 = 15;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int LIMIT_DISTANCE = 25; // Limit in cm

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);

  // Set motor speed pins HIGH (full speed)
  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, HIGH);
  
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

  if (distance < 0 || distance > 150) {
    distance = 150;
  }

  // Draw HUD on OLED
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(15, 3);
  display.print("NAV DASHBOARD");

  // Display distance and status
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 20);
  display.print("Obstacle: ");
  display.print(distance);
  display.print(" cm");

  display.setCursor(10, 32);

  // Collision detection and steering logic
  if (distance > 0 && distance < LIMIT_DISTANCE) {
    // Obstacle detected - stop and steer away
    display.print("ACTION  : !! REVERTING !!");
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    stopRobot();
    delay(300);
    moveBackward();
    delay(800);
    stopRobot();
    delay(200);
    turnRight();
    delay(600);
    stopRobot();
    delay(200);
  } else {
    // Clear path - drive forward
    display.print("ACTION  : FORWARD");
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
    
    moveForward();
  }

  delay(60); // Settle delay
}

void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

void turnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left forward
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); // Right backward
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-SR04 Sensor**, **SSD1306 OLED**, **L298N**, and **two DC Motors** onto the canvas.
2. Connect HC-SR04 to **GP16/GP17**, OLED to **GP4/GP5**, and L298N control to **GP10-15**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the distance slider below 25 cm to simulate an obstacle, and watch the wheels reverse and the OLED display change.

## Expected Output

Terminal:
```
Simulation active. Obstacle avoider steer loops with OLED HUD active.
```

## Expected Canvas Behavior
* Clear Path (Distance > 30 cm): OLED reads `ACTION  : FORWARD`. Both wheels spin forward.
* Obstacle Detected (Distance < 25 cm): OLED displays `ACTION  : !! REVERTING !!`, wheels reverse and spin in opposite directions to turn the robot.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.print("ACTION  : !! REVERTING !!")` | Updates the navigation screen with the active steering maneuver when an obstacle is detected. |

## Hardware & Safety Concept: Real-Time Telemetry Displays
Adding telemetry displays to autonomous mobile robots helps engineers debug and monitor navigation code. By showing sensor readings (distance) and active steering commands (FORWARD, REVERTING) on an onboard OLED, you can verify that the robot's sensors and motors are responding correctly in real time without needing to connect a debug cable.

## Try This! (Challenges)
1. **Audio Alarm**: Connect a buzzer on GP14 and sound a pulsing warning beep while the robot is reversing.
2. **Dynamic Alert LEDs**: Flash warning lights on GP15 when an obstacle is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED display is garbled or text overlaps | Missing screen clear | Ensure `display.clearDisplay()` is executed at the start of each display loop cycle before printing new text. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [114 - Pico Obstacle Avoider](../intermediate/114-pico-obstacle-avoider.md)
- [118 - Pico Ultrasonic OLED](../intermediate/118-pico-ultrasonic-oled.md)
- [142 - Pico BT Robot Speed](142-pico-bt-robot-speed.md)
