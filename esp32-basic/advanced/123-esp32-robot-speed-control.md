# 123 - ESP32 Robot Speed Control

Control a mobile robot's acceleration and speed using the ESP32's LEDC hardware PWM peripheral to drive both L298N enable channels independently.

## Goal
Learn how to configure dual-channel LEDC hardware PWM to scale the speeds of left and right motors concurrently, avoiding software timing bottlenecks.

## What You Will Build
An L298N motor driver controls two DC motors. ENA (Left Motor) connects to GPIO 5 (LEDC Channel 0) and ENB (Right Motor) to GPIO 23 (LEDC Channel 1). The robot ramps speed from 30% to 60%, then to 100% forward, and ramps back down.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motors (2) | `dc_motor` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 | GPIO18 / GPIO19 | Yellow / Green | Left direction bits |
| L298N Module | ENA | GPIO5 | Orange | Left speed PWM |
| L298N Module | IN3 / IN4 | GPIO21 / GPIO22 | Blue / Purple | Right direction bits |
| L298N Module | ENB | GPIO23 | White | Right speed PWM |
| L298N Module | GND | GND | Black | Common Ground |
| L298N Module | 12V (VCC) | External battery + | Red | Motor power |
| L298N Module | GND (power) | External battery − | Black | Power ground |
| Left DC Motor | Terminals | OUT1 / OUT2 | Blue / White | Left motor |
| Right DC Motor | Terminals | OUT3 / OUT4 | Blue / White | Right motor |

> **Wiring tip:** Connect ENA to GPIO 5 and ENB to GPIO 23. This project uses hardware PWM, so make sure these pins are wired directly without jumpers.

## Code
```cpp
// Robot Speed Control (PWM scaling)
const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;

const int IN3 = 21;
const int IN4 = 22;
const int ENB_PIN = 23;

// PWM Configuration
const int PWM_FREQ = 1000;
const int PWM_RES = 8; // 8-bit: 0-255

// Allocate two independent LEDC channels
const int LEDC_CHAN_LEFT = 0;
const int LEDC_CHAN_RIGHT = 1;

void setRobotSpeed(int leftDuty, int rightDuty) {
  // Clamp values for safety
  leftDuty = constrain(leftDuty, 0, 255);
  rightDuty = constrain(rightDuty, 0, 255);
  
  ledcWrite(LEDC_CHAN_LEFT, leftDuty);
  ledcWrite(LEDC_CHAN_RIGHT, rightDuty);
  
  int leftPct = leftDuty * 100 / 255;
  int rightPct = rightDuty * 100 / 255;
  
  Serial.print("Left Speed: "); Serial.print(leftPct);
  Serial.print("%  |  Right Speed: "); Serial.print(rightPct);
  Serial.println("%");
}

void robotForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void robotStop() {
  setRobotSpeed(0, 0);
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT); pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT); pinMode(IN4, OUTPUT);
  
  // Set up Left channel
  ledcSetup(LEDC_CHAN_LEFT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, LEDC_CHAN_LEFT);
  
  // Set up Right channel
  ledcSetup(LEDC_CHAN_RIGHT, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENB_PIN, LEDC_CHAN_RIGHT);
  
  robotStop();
  robotForward(); // Set direction bits forward
  
  Serial.println("Robot Speed Controller Initialised.");
}

void loop() {
  // 1. Slow Speed (30%)
  Serial.println("--- Speed: 30% ---");
  setRobotSpeed(76, 76); // 255 * 0.3 = ~76
  delay(2000);
  
  // 2. Medium Speed (60%)
  Serial.println("--- Speed: 60% ---");
  setRobotSpeed(153, 153); // 255 * 0.6 = ~153
  delay(2000);
  
  // 3. Full Speed (100%)
  Serial.println("--- Speed: 100% ---");
  setRobotSpeed(255, 255);
  delay(2000);
  
  // 4. Decelerate to 30%
  Serial.println("--- Decelerating ---");
  setRobotSpeed(76, 76);
  delay(2000);
  
  // 5. Stop
  Serial.println("--- Stopped ---");
  robotStop();
  delay(2000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, and two **DC Motors** onto the canvas.
2. Wire ENA to **GPIO5**, ENB to **GPIO23**, and inputs 1–4 to **GPIO18, 19, 21, 22**.
3. Paste the code and click **Run**.
4. Observe the motor rotation speed changes during each step of the cycle on the canvas.

## Expected Output
Serial Monitor:
```
Robot Speed Controller Initialised.
--- Speed: 30% ---
Left Speed: 29%  |  Right Speed: 29%
--- Speed: 60% ---
Left Speed: 60%  |  Right Speed: 60%
--- Speed: 100% ---
Left Speed: 100%  |  Right Speed: 100%
```

## Expected Canvas Behavior
* Both motors spin forward slowly (30%), speed up to moderate (60%), spin at maximum speed (100%), slow back down, and stop.
* The cycle repeats automatically.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `ledcSetup(LEDC_CHAN_LEFT, ...)` | Allocates hardware PWM channel 0 at 1 kHz frequency. |
| `ledcAttachPin(ENA_PIN, ...)` | Binds the Left enable pin (GPIO 5) to channel 0. |
| `ledcWrite(LEDC_CHAN_RIGHT, rightDuty)` | Updates the PWM duty cycle for the Right enable pin. |

## Hardware & Safety Concept: Hardware PWM Timers and Speed Calibration
Hardware PWM controllers run independently of the main CPU cores. Once configured (`ledcWrite`), the ESP32 hardware timer maintains the duty cycle pulse ratio without needing code execution, ensuring stable motor speeds even if the main loop pauses or handles slow sensor reads. Differential robots require independent left and right speed control to compensate for motor manufacturing variances (one wheel turning slightly faster than the other).

## Try This! (Challenges)
1. **Differential Speed steering**: Write a function `steerLeft(int turnPct)` that reduces the Left motor speed by a percentage while maintaining full speed on the Right motor.
2. **Potentiometer Dual Speed dial**: Read a potentiometer (GPIO 34) and map it to set the master speed of the robot.
3. **Smooth Speed ramp**: Replace the step delays with a loop that increments speed duty by 1 every 15 ms to achieve smooth acceleration.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One motor does not change speed | Wrong LEDC channel mapping | Ensure `LEDC_CHAN_LEFT` and `LEDC_CHAN_RIGHT` use different channel indices (e.g. 0 and 1) |
| Motors squeal but don't spin | PWM frequency is too high | Lower `PWM_FREQ` to 500 Hz or 1 kHz to allow coils to magnetize |
| Speed is erratic | Missing ground reference | Verify all GND pins are tied to a common rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - ESP32 Dual Motor Forward/Reverse Drive](121-esp32-dual-motor-forward-reverse-drive.md)
- [122 - ESP32 Mobile Robot Left/Right Steering](122-esp32-mobile-robot-left-right-steering.md)
- [54 - ESP32 DC Motor Speed Scaling](../intermediate/54-esp32-dc-motor-speed-scaling.md)
