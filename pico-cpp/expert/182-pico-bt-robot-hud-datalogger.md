# 182 - Pico BT Robot HUD Datalogger

Build a wireless mobile robot chassis that steers via Bluetooth, avoids obstacles autonomously, and streams speed and distance telemetry logs back to the host.

## Goal
Learn how to parse Bluetooth commands (UART), interface HC-SR04 ultrasonic rangefinders, drive DC motors with variable speed (PWM), and stream CSV log packets.

## What You Will Build
An autonomous and remote-controlled hybrid vehicle:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives manual steering commands.
- **HC-SR04 Sensor (Trig GP16, Echo GP17)**: Measures front obstacle distance.
- **L298N & Dual DC Motors (GP10-15)**: Drives the robot.
- **16x2 I2C LCD (GP4, GP5)**: Displays active speed levels and obstacle distances.
- **Bluetooth Telemetry**: Streams CSV logs (e.g. `Speed,Distance_cm,Action_code`) every second.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors | `dc_motor` | Yes (two motors) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 lines |
| HC-SR04 | Trig / Echo | GP16 / GP17 | Rangefinder lines |
| L298N Control | ENA / ENB | GP10 / GP11 | Left/Right speed PWM |
| L298N Control | IN1 / IN2 | GP12 / GP13 | Left motor direction |
| L298N Control | IN3 / IN4 | GP14 / GP15 | Right motor direction |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int TRIG_PIN = 16;
const int ECHO_PIN = 17;

const int ENA = 10; const int ENB = 11;
const int IN1 = 12; const int IN2 = 13;
const int IN3 = 14; const int IN4 = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);

int currentSpeed = 160; // Default speed
const int OBSTACLE_LIMIT = 25; // Stop threshold in cm

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);

  pinMode(ENA, OUTPUT); pinMode(ENB, OUTPUT);
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);

  // Apply default speed
  analogWrite(ENA, currentSpeed);
  analogWrite(ENB, currentSpeed);

  digitalWrite(TRIG_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Robot Console");
  lcd.setCursor(0, 1);
  lcd.print("Telemetry Online");

  // Print CSV Header to Bluetooth
  Serial1.println("Speed_pct,Distance_cm,Action");
  delay(1000);
}

void loop() {
  // 1. Measure obstacle distance
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  int distance = duration * 0.0343 / 2;

  if (distance < 0 || distance > 150) {
    distance = 150;
  }

  int actionCode = 0; // 0 = Stop/AutoStop, 1 = Manual Forward, 2 = Manual Rev

  // 2. Obstacle Override Protection
  if (distance > 0 && distance < OBSTACLE_LIMIT) {
    stopRobot();
    actionCode = 0;
    Serial1.println("CRITICAL: Obstacle detected! Auto-stop activated.");
  } else {
    // 3. Process Bluetooth Manual commands if path is clear
    if (Serial1.available()) {
      char cmd = Serial1.read();

      if (cmd == 'F') {
        moveForward();
        actionCode = 1;
      } 
      else if (cmd == 'B') {
        moveBackward();
        actionCode = 2;
      } 
      else if (cmd == 'S') {
        stopRobot();
        actionCode = 0;
      }
      else if (cmd == '1') {
        currentSpeed = 100;
        analogWrite(ENA, currentSpeed);
        analogWrite(ENB, currentSpeed);
      }
      else if (cmd == '2') {
        currentSpeed = 180;
        analogWrite(ENA, currentSpeed);
        analogWrite(ENB, currentSpeed);
      }
      else if (cmd == '3') {
        currentSpeed = 255;
        analogWrite(ENA, currentSpeed);
        analogWrite(ENB, currentSpeed);
      }
    }
  }

  // 4. Update LCD
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Dist : ");
  lcd.print(distance);
  lcd.print(" cm");

  lcd.setCursor(0, 1);
  lcd.print("Speed: ");
  lcd.print(currentSpeed * 100 / 255);
  lcd.print("% Mode:");
  lcd.print(actionCode);

  // 5. Stream telemetry logs over Bluetooth
  Serial1.print(currentSpeed * 100 / 255);
  Serial1.print(",");
  Serial1.print(distance);
  Serial1.print(",");
  Serial1.println(actionCode);

  delay(200);
}

void moveForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
}

void stopRobot() {
  digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05**, **HC-SR04 Sensor**, **L298N**, **two DC Motors**, and **I2C LCD** onto the canvas.
2. Connect HC-05 to **GP0/GP1**, HC-SR04 to **GP16/GP17**, L298N to **GP10-15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'F'` in the Bluetooth terminal to spin the wheels, and slide the distance sensor to test the automatic stop override.

## Expected Output

Terminal (Bluetooth):
```
Speed_pct,Distance_cm,Action
62,150,1
CRITICAL: Obstacle detected! Auto-stop activated.
0,15,0
```

## Expected Canvas Behavior
* Press `'F'` on BT: Both wheels spin forward. LCD reads `Speed: 62% Mode:1`.
* Slide distance slider < 25 cm: Both wheels stop spinning immediately. LCD reads `Mode:0`. Bluetooth prints `CRITICAL: Obstacle detected!`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `distance < OBSTACLE_LIMIT` | Distance threshold checker that overrides manual drive commands and stops the motors if a collision risk is detected. |

## Hardware & Safety Concept: Collision Avoidance Logic
Mobile robots combine remote manual commands (such as Bluetooth steering) with autonomous override loops (such as ultrasonic distance braking). If the manual driver guides the robot toward a wall, the rangefinder override takes control and cuts motor power, preventing damage from collisions.

## Try This! (Challenges)
1. **Warning LED**: Flash a warning LED on GP8 when an obstacle is detected.
2. **Reverse Steer**: Configure the robot to automatically reverse and turn away when an obstacle blocks its path.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors hum but do not spin | PWM drive too low | DC motors need a minimum startup voltage (usually >35% duty cycle) to spin. Increase the low-speed PWM limit. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [112 - Pico Bluetooth Robot](../intermediate/112-pico-bt-robot.md)
- [114 - Pico Obstacle Avoider](../intermediate/114-pico-obstacle-avoider.md)
- [142 - Pico BT Robot Speed](../advanced/142-pico-bt-robot-speed.md)
- [144 - Pico Obstacle Avoider OLED](../advanced/144-pico-obstacle-avoider-oled.md)
