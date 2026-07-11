# 130 - ESP32 Light Seeking Robotic Bug

Build a light-seeking mobile robot (Braitenberg Vehicle) that reads two front-mounted LDR sensors and steers towards the brightest light source using differential motor control.

## Goal
Learn how to implement phototaxis behavior, compare multiple analog sensor inputs, and translate differences into directional steering commands.

## What You Will Build
An L298N driver controls two DC motors (Left: 18/19/5, Right: 21/22/23). Two LDR sensors are mounted pointing forward-left (GPIO 34) and forward-right (GPIO 35). The robot compares light levels. If the left sensor reads brighter, the robot steers left; if the right sensor reads brighter, it steers right. If both read similar bright levels, it drives straight. If it is dark, it stops.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| LDRs (Photoresistors) (2) | `ldr` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Left Motor |
| L298N Module | IN3 / IN4 / ENB | GPIO21 / GPIO22 / GPIO23 | Blue/Purple/White | Right Motor |
| Left LDR | Leg 1 / Leg 2 | 3V3 / GPIO34 | Red / Yellow | Divider top |
| 10 kΩ Resistor Left | Leg 1 / Leg 2 | GPIO34 / GND | White / Black | Divider bottom |
| Right LDR | Leg 1 / Leg 2 | 3V3 / GPIO35 | Red / Yellow | Divider top |
| 10 kΩ Resistor Right| Leg 1 / Leg 2 | GPIO35 / GND | White / Black | Divider bottom |
| L298N Module | GND | GND | Black | Shared logic ground |

> **Wiring tip:** Set up two identical voltage divider circuits for the LDRs on GPIO 34 and 35. Make sure the LDRs are physically separated or shielded with a vertical barrier between them to ensure light differential readings.

## Code
```cpp
// Light Seeking Robotic Bug
const int IN1 = 18;
const int IN2 = 19;
const int ENA = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB = 23;

const int LDR_LEFT = 34;
const int LDR_RIGHT = 35;

// Light difference threshold to trigger steering (raw ADC counts)
const int STEER_THRESHOLD = 300; 

// Ambient darkness threshold: if both sensors read below this, stop
const int DARKNESS_LIMIT = 500; 

void robotForward() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

void robotStop() {
  digitalWrite(ENA, LOW);
  digitalWrite(ENB, LOW);
}

// Gentle turn left (run right wheel, stop left)
void robotTurnLeft() {
  digitalWrite(ENA, LOW);
  digitalWrite(IN3, HIGH); digitalWrite(IN4, LOW);  digitalWrite(ENB, HIGH);
}

// Gentle turn right (run left wheel, stop right)
void robotTurnRight() {
  digitalWrite(IN1, HIGH); digitalWrite(IN2, LOW);  digitalWrite(ENA, HIGH);
  digitalWrite(ENB, LOW);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT); pinMode(ENA, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT); pinMode(ENB, OUTPUT);
  
  robotStop();
  Serial.println("Light Seeking Bug ready.");
}

void loop() {
  int ldrL = analogRead(LDR_LEFT);
  int ldrR = analogRead(LDR_RIGHT);
  
  Serial.print("LDR Left: "); Serial.print(ldrL);
  Serial.print(" | LDR Right: "); Serial.println(ldrR);
  
  // 1. Darkness Safe Mode
  // If both sensors read dark, sleep/stop to conserve power
  if (ldrL < DARKNESS_LIMIT && ldrR < DARKNESS_LIMIT) {
    Serial.println("State: SLEEPING (Too dark)");
    robotStop();
  } 
  // 2. Steer Left (left is brighter than right + threshold)
  else if (ldrL > ldrR + STEER_THRESHOLD) {
    Serial.println("State: STEER LEFT");
    robotTurnLeft();
  } 
  // 3. Steer Right (right is brighter than left + threshold)
  else if (ldrR > ldrL + STEER_THRESHOLD) {
    Serial.println("State: STEER RIGHT");
    robotTurnRight();
  } 
  // 4. Balanced Light -> Drive Forward
  else {
    Serial.println("State: FORWARD");
    robotForward();
  }
  
  delay(100); // 10Hz evaluation rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, two **DC Motors**, and two **LDRs** onto the canvas.
2. Wire Left LDR to **GPIO34**, Right LDR to **GPIO35**, and motors to **18, 19, 5, 21, 22, 23**.
3. Paste the code and click **Run**.
4. Adjust LDR light levels. Slide Left LDR higher than Right LDR. Watch the Left wheel stop and Right wheel spin.
5. Set both LDRs low. Watch the robot stop.

## Expected Output
Serial Monitor:
```
Light Seeking Bug ready.
LDR Left: 3200 | LDR Right: 3100
State: FORWARD
LDR Left: 3400 | LDR Right: 1200
State: STEER LEFT
LDR Left: 350 | LDR Right: 380
State: SLEEPING (Too dark)
```

## Expected Canvas Behavior
* Both LDR sliders bright: Both motors spin clockwise.
* Left LDR slider bright, Right LDR slider dark: Left motor stops, Right motor spins (turns left).
* Right LDR slider bright, Left LDR slider dark: Left motor spins, Right motor stops (turns right).
* Both sliders dragged to dark: Both motors stop.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ldrL > ldrR + STEER_THRESHOLD` | Evaluates if the Left LDR is significantly brighter than the Right LDR. |
| `ldrL < DARKNESS_LIMIT` | Detects when the environment goes dark to sleep the motors. |
| `robotTurnLeft()` | Shuts off the Left wheel to pivot the robot Left towards the light source. |

## Hardware & Safety Concept: Braitenberg Vehicles and Phototaxis
A Braitenberg Vehicle uses direct sensor-to-actuator connections to produce complex behaviors. Wiring a light sensor to the opposite motor (Left sensor controls Right wheel, Right sensor controls Left wheel) creates a light-seeking robot (phototaxis). If the left sensor sees light, the right wheel spins, turning the robot left towards the light until both sensors receive equal light and the robot drives straight.

## Try This! (Challenges)
1. **Light Fleeing (Shadow Bug)**: Swap the outputs in code so the robot avoids light and seeks shade (photophobia).
2. **Proportional Speed seeking**: Use PWM speed control (Project 123) to make the motors turn faster when the light source is brighter.
3. **Battery Low Check**: Add an LED indicator that flashes if LDR readings are extremely low for 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Robot turns away from flashlights | Steering functions reversed | Swap the `robotTurnLeft()` and `robotTurnRight()` function calls in the code |
| Robot wiggles rapidly back and forth | `STEER_THRESHOLD` is set too low | Increase the threshold (e.g. 500) to ignore minor differences |
| Robot does not respond to light | Resistors wired incorrectly | Verify pull-down resistors are tied between the GPIO pins and GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - ESP32 LDR Light Intensity Meter](../beginner/33-esp32-ldr-light-intensity-meter.md)
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [129 - ESP32 Robot Wall Follower](129-esp32-robot-wall-follower.md)
