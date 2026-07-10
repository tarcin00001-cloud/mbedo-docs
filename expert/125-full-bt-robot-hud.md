# 125 - Full BT Robot HUD

Construct a Bluetooth-controlled mobile robot utilizing an L298N motor driver, an HC-05 serial link, an HC-SR04 ultrasonic crash prevention sensor, and an onboard I2C LCD displaying live navigation telemetry.

## Goal
Learn how to design a complex mobile vehicle system that integrates serial control parsers, real-time safety distance calculations, and output coordination (motor drivers, display output) without blocking execution in interpreted mode.

## What You Will Build
A smart telemetry-guided robotic vehicle:
- **Bluetooth Navigation**: Receives control keys over HC-05:
  - `F` / `f`: Move forward
  - `B` / `b`: Move backward
  - `L` / `l`: Turn left
  - `R` / `r`: Turn right
  - `S` / `s`: Stop
- **Collision Avoidance**: If distance reads below 20 cm, the vehicle automatically halts and overrides any forward commands.
- **Onboard HUD LCD**: Prints the current speed, direction vector, and distance metrics.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| HC-05 Bluetooth | `hc05_bluetooth` | Yes | Yes |
| HC-SR04 Ultrasonic | `hc_sr04` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| DC Motors (Pair) | `dc_motor` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| HC-05 Bluetooth | VCC | 5V | Power supply |
| HC-05 Bluetooth | TXD | D10 | SoftwareSerial RX (Ardu RX ← HC TX) |
| HC-05 Bluetooth | RXD | D11 | SoftwareSerial TX (Ardu TX → HC RX) |
| HC-05 Bluetooth | GND | GND | Ground reference |
| HC-SR04 Sensor | VCC | 5V | Power supply |
| HC-SR04 Sensor | TRIG | D2 | Trigger output pin |
| HC-SR04 Sensor | ECHO | D3 | Echo input pin |
| HC-SR04 Sensor | GND | GND | Ground reference |
| L298N Driver | IN1 | D4 | Motor Left Direction 1 |
| L298N Driver | IN2 | D5 | Motor Left Direction 2 |
| L298N Driver | IN3 | D6 | Motor Right Direction 1 |
| L298N Driver | IN4 | D7 | Motor Right Direction 2 |
| L298N Driver | ENA | D8 | Motor Left Speed (tie HIGH/PWM) |
| L298N Driver | ENB | D9 | Motor Right Speed (tie HIGH/PWM) |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <SoftwareSerial.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int BT_RX      = 10;
const int BT_TX      = 11;
const int TRIG_PIN   = 2;
const int ECHO_PIN   = 3;

// Motor pins
const int IN1 = 4;
const int IN2 = 5;
const int IN3 = 6;
const int IN4 = 7;
const int ENA = 8;
const int ENB = 9;

SoftwareSerial BT(BT_RX, BT_TX);
LiquidCrystal_I2C lcd(0x27, 16, 2);

char currentDir = 'S'; // Default Stopped
float distance = 100.0;

void setup() {
  Serial.begin(9600);
  BT.begin(9600);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENA, OUTPUT);
  pinMode(ENB, OUTPUT);

  // Enable speed controllers
  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, HIGH);

  // Default stop
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Robot HUD Act.");
  delay(1500);
  lcd.clear();

  Serial.println("Robot Controller Online");
}

void loop() {
  // 1. Measure obstacle distance
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);

  long duration = pulseIn(ECHO_PIN, HIGH);
  distance = duration * 0.034 / 2.0;

  // 2. Parse incoming Bluetooth commands
  if (BT.available()) {
    char cmd = BT.read();
    
    if (cmd == 'F' || cmd == 'f') { currentDir = 'F'; }
    else if (cmd == 'B' || cmd == 'b') { currentDir = 'B'; }
    else if (cmd == 'L' || cmd == 'l') { currentDir = 'L'; }
    else if (cmd == 'R' || cmd == 'r') { currentDir = 'R'; }
    else if (cmd == 'S' || cmd == 's') { currentDir = 'S'; }
  }

  // 3. Collision safety override
  if (distance > 0 && distance < 20.0) {
    if (currentDir == 'F') {
      currentDir = 'S'; // Force stop if heading toward wall
      Serial.println("Safety Stop Triggered!");
      BT.println("ALERT: Obstacle Stop!");
    }
  }

  // 4. Drive L298N motors based on direction
  if (currentDir == 'F') {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);
  } else if (currentDir == 'B') {
    digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH);
    digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH);
  } else if (currentDir == 'L') {
    digitalWrite(IN1, LOW);  digitalWrite(IN2, HIGH); // Left spins back
    digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  // Right spins forward
  } else if (currentDir == 'R') {
    digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  // Left spins forward
    digitalWrite(IN3, LOW);  digitalWrite(IN4, HIGH); // Right spins back
  } else { // 'S' / Stop
    digitalWrite(IN1, LOW);  digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);  digitalWrite(IN4, LOW);
  }

  // 5. Update HUD LCD Display
  // Line 1: Nav status
  lcd.setCursor(0, 0);
  lcd.print("DIR: ");
  if (currentDir == 'F')      { lcd.print("FORWARD  "); }
  else if (currentDir == 'B') { lcd.print("BACKWARD "); }
  else if (currentDir == 'L') { lcd.print("LEFT TURN"); }
  else if (currentDir == 'R') { lcd.print("RIGHT TRN"); }
  else                        { lcd.print("STOPPED  "); }

  // Line 2: Distance feedback
  lcd.setCursor(0, 1);
  lcd.print("RANGE: ");
  lcd.print(distance, 0);
  lcd.print(" cm    ");

  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **HC-05 Bluetooth**, **HC-SR04 Ultrasonic**, **L298N Motor Driver**, **DC Motors**, and **16x2 I2C LCD** onto the canvas.
2. Connect HC-05: **VCC** to **5V**, **TXD** to **D10**, **RXD** to **D11**, **GND** to **GND**.
3. Connect HC-SR04: **VCC** to **5V**, **TRIG** to **D2**, **ECHO** to **D3**, **GND** to **GND**.
4. Connect L298N: **IN1..4** to **D4..7**, **ENA/ENB** to **D8..9**, VCC/GND to power rails.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste code, select the interpreted mode, and click **Run**.
7. In the **Bluetooth Terminal** tab, send navigation command characters (e.g. `F`, `L`, `S`).
8. Decrease the ultrasonic sensor slider distance below 20 cm while moving forward to watch the safety stop function override motor activity.

## Expected Output

Terminal:
```
Robot Controller Online
Safety Stop Triggered!
```

Bluetooth Terminal:
```
ALERT: Obstacle Stop!
```

LCD Display:
```
DIR: STOPPED  
RANGE: 15 cm    
```

## Expected Canvas Behavior
| BT Command Input | Distance (HC-SR04) | Motor Direction Pins | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- |
| `'S'` | 120 cm | All LOW | `DIR: STOPPED` | `RANGE: 120 cm` |
| `'F'` | 120 cm | IN1=HIGH, IN3=HIGH | `DIR: FORWARD` | `RANGE: 120 cm` |
| `'F'` | 15 cm | All LOW (Overridden) | `DIR: STOPPED` | `RANGE: 15 cm` |
| `'L'` | 15 cm | IN2=HIGH, IN3=HIGH | `DIR: LEFT TURN` | `RANGE: 15 cm` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `currentDir = 'S'` | Overrides navigation state immediately, bypassing keyboard inputs to prevent crashes. |
| `digitalWrite(ENA, HIGH)` | Applies high speed/enable logic to the left side motor array (direct 5V equivalent logic). |
| `lcd.print(distance, 0)` | Displays distance integer value omitting trailing decimal details to keep visual HUD layouts aligned. |

## Hardware & Safety Concept: Collision Override
Mobile robotic vehicles require local, fail-safe override systems. Standard wireless control modules are prone to packet drops or communication latency. If a robot relies purely on Bluetooth command feedback to halt, a connection drop will cause it to crash. Local ultrasonic sensors act as hardware interrupts, overriding commands locally when hazard thresholds are breached.

## Try This! (Challenges)
1. **Dynamic Brake Lights**: Wire a red LED to pin D12 and program it to turn ON only when safety stop overrides the forward navigation movement.
2. **Telemetry Broadcast**: Stream speed and range data back to the Bluetooth controller terminal every 1.5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motors spin opposite way | Left/right coils swapped | Switch the direction pins or swap the positive/negative output cables leading to the driver. |
| LCD freezes when motors run | Ground noise interference | Connect a separate power battery to L298N driver and share only common grounds with the Arduino. |

## Mode Notes
This multi-module navigation and telemetry system runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [84 - Robot Forward/Back](../intermediate/84-robot-forward-back.md)
- [85 - BT Remote Car](../intermediate/85-bt-remote-car.md)
- [114 - BT Robot with Sensor](../advanced/114-bt-robot-with-sensor.md)
