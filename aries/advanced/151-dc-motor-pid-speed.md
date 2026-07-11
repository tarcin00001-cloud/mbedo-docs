# 151 - DC Motor PID Speed Controller

Regulate the rotational speed of a DC motor using PID calculations based on encoder feedback pulses.

## Goal
Learn how to implement a closed-loop Proportional-Integral-Derivative (PID) speed controller using state variables without loops, measuring encoder transitions and dynamically adjusting PWM output.

## What You Will Build
A closed-loop speed regulator. The encoder outputs pulses on GPIO 14 and GPIO 15 as the motor spins. The code polls these pin states, calculates the speed (RPM), compares it to the target speed, computes the PID error correction, and outputs a corrected PWM duty cycle to the L298N motor driver to maintain constant speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DC Motor with Encoder | `motor_encoder` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | ENA | GPIO 9 | Orange | PWM speed control signal |
| L298N Driver | IN1 | GPIO 7 | Yellow | Motor direction pin 1 |
| L298N Driver | IN2 | GPIO 8 | Green | Motor direction pin 2 |
| L298N Driver | VCC | 5V | Red | Power input |
| L298N Driver | GND | GND | Black | Ground connection |
| DC Motor | Terminal A | OUT1 | Red | Output from driver |
| DC Motor | Terminal B | OUT2 | Black | Output from driver |
| Encoder | VCC | 3V3 | Red | Encoder power |
| Encoder | GND | GND | Black | Encoder ground |
| Encoder | OUT A | GPIO 14 | Blue | Primary pulse counter |
| Encoder | OUT B | GPIO 15 | White | Quadrature direction decoder |

> **Wiring tip:** Double-check that your DC motor's encoder pins are connected to GPIO 14 and 15. The encoder requires 3.3V logic levels, which are native to the VEGA ARIES v3 board.

## Code
```cpp
// DC Motor PID Speed Controller - VEGA ARIES v3
const int ENA_PIN = 9;
const int IN1_PIN = 7;
const int IN2_PIN = 8;
const int ENCODER_A = 14;
const int ENCODER_B = 15;

int lastEncoderA = LOW;
long pulseCount = 0;
unsigned long lastTime = 0;
float targetRPM = 120.0;
float currentRPM = 0.0;

// PID Tuning Parameters
float Kp = 1.5;
float Ki = 0.8;
float Kd = 0.1;

float integral = 0.0;
float lastError = 0.0;
int pwmOutput = 0;

void setup() {
  Serial.begin(115200);
  pinMode(ENA_PIN, OUTPUT);
  pinMode(IN1_PIN, OUTPUT);
  pinMode(IN2_PIN, OUTPUT);
  pinMode(ENCODER_A, INPUT);
  pinMode(ENCODER_B, INPUT);

  // Set motor to spin forward
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, LOW);

  lastEncoderA = digitalRead(ENCODER_A);
  lastTime = millis();
}

void loop() {
  int stateA = digitalRead(ENCODER_A);
  
  // Detect state change on Encoder A
  if (stateA != lastEncoderA) {
    if (stateA == HIGH) {
      int stateB = digitalRead(ENCODER_B);
      if (stateB == LOW) {
        pulseCount = pulseCount + 1;
      } else {
        pulseCount = pulseCount - 1;
      }
    }
    lastEncoderA = stateA;
  }

  unsigned long currentTime = millis();
  unsigned long elapsed = currentTime - lastTime;

  // Calculate and regulate speed every 100ms
  if (elapsed >= 100) {
    // RPM calculation assuming 20 pulses per revolution
    float pulsesPerSec = (float)pulseCount / (float)elapsed * 1000.0;
    currentRPM = (pulsesPerSec * 60.0) / 20.0;
    pulseCount = 0;
    lastTime = currentTime;

    // PID Calculations
    float error = targetRPM - currentRPM;
    integral = integral + (error * ((float)elapsed / 1000.0));
    float derivative = (error - lastError) / ((float)elapsed / 1000.0);
    lastError = error;

    float pidVal = (Kp * error) + (Ki * integral) + (Kd * derivative);
    pwmOutput = (int)pidVal;

    // Constrain PWM output to 8-bit limit
    if (pwmOutput < 0) {
      pwmOutput = 0;
    }
    if (pwmOutput > 255) {
      pwmOutput = 255;
    }

    analogWrite(ENA_PIN, pwmOutput);

    Serial.print("Target: ");
    Serial.print(targetRPM);
    Serial.print(" RPM | Actual: ");
    Serial.print(currentRPM);
    Serial.print(" RPM | PWM Output: ");
    Serial.println(pwmOutput);
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **L298N Motor Driver**, and **DC Motor with Encoder** onto the canvas.
2. Connect the L298N Driver: **ENA** to **GPIO 9**, **IN1** to **GPIO 7**, **IN2** to **GPIO 8**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the Motor outputs **OUT1** and **OUT2** of the L298N driver to the **Terminal A** and **Terminal B** of the DC Motor.
4. Connect the Motor Encoder: **VCC** to **3V3**, **GND** to **GND**, **OUT A** to **GPIO 14**, and **OUT B** to **GPIO 15**.
5. Copy the project code and paste it into the editor.
6. Click **Run**.
7. Observe the motor spinning and regulating its speed to the target RPM in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Target: 120.00 RPM | Actual: 0.00 RPM | PWM Output: 180
Target: 120.00 RPM | Actual: 85.00 RPM | PWM Output: 232
Target: 120.00 RPM | Actual: 118.50 RPM | PWM Output: 215
Target: 120.00 RPM | Actual: 120.20 RPM | PWM Output: 210
```

## Expected Canvas Behavior
* The motor starts rotating, accelerating toward its target.
* Under changing motor loads (if simulated), the PWM output dynamically adjusts to keep the motor's speed stable at 120 RPM.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `stateA != lastEncoderA` | Detects transition edges on encoder pin A to count rotational ticks. |
| `pulseCount = pulseCount + 1` | Increments pulse count when Encoder B lags A, indicating forward direction. |
| `elapsed >= 100` | Limits speed calculation updates to every 100 milliseconds. |
| `currentRPM = ...` | Scales counted pulses to revolutions per minute based on the motor's CPR. |
| `error = targetRPM - currentRPM` | Determines the difference between desired speed and measured speed. |
| `pidVal = (Kp * error) ...` | Calculates corrective action combining proportional, integral, and derivative terms. |
| `analogWrite(ENA_PIN, pwmOutput)` | Applies the calculated duty cycle to the L298N PWM speed controller. |

## Hardware & Safety Concept
* **Optocoupler / Hall Effect Encoders**: High-speed DC motors rely on either optical or magnetic encoders. Ensure the encoder sensor's ground is securely tied to the ARIES board ground to prevent electrical noise from causing false edge-trigger counts.
* **Flyback Protection**: L298N modules have built-in freewheeling diodes. On custom boards, always include these diodes to protect switching transistors against inductive kickback.

## Try This! (Challenges)
1. **Dynamic Setpoint**: Use a potentiometer on `ADC0` to set the target RPM dynamically between 0 and 200 RPM.
2. **PID Tuner**: Adjust `Kp` to 3.0 and `Ki` to 0.0 and observe how speed oscillates or takes longer to settle on a target speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor spins at maximum speed and PID does not regulate | Encoder wires swapped or disconnected | Verify OUT A and OUT B connections; if the direction is decoded backwards, the error integral will run away to maximum. |
| Motor hums but does not spin | Target RPM too low / Ki too small | Increase target RPM or increase Ki parameter to accumulate error and break static friction. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - DC Motor Speed Scaling](../intermediate/54-dc-motor-speed-scaling.md)
- [152 - Servo PID Angle Balancer](152-servo-pid-angle-balancer.md)
