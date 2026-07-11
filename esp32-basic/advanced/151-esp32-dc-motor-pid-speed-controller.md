# 151 - ESP32 DC Motor PID Speed Controller

Build a closed-loop speed control system that measures DC motor RPM using a rotary encoder sensor, compares it against a target speed (set by a potentiometer), and calculates PID feedback corrections to maintain speed under variable loads.

## Goal
Learn how to count encoder pulses using GPIO interrupts, compute rotational speed (RPM), and implement a Proportional-Integral-Derivative (PID) control algorithm.

## What You Will Build
An L298N driver controls a DC motor (IN1: 18, IN2: 19, ENA: 5). A rotary encoder disk and photo-interrupter sensor are mounted on the motor shaft, outputting pulses to GPIO 4 (interrupt pin). A potentiometer on GPIO 34 sets the target speed (0 to 150 RPM). The ESP32 runs a PID loop to adjust the PWM duty cycle to maintain the target RPM even when mechanical friction increases.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `motor_driver` | Yes | Yes |
| DC Motor with Encoder | `encoder_motor` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| External Power Supply (6–12 V) | — | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Module | IN1 / IN2 / ENA | GPIO18 / GPIO19 / GPIO5 | Yellow/Green/Orange | Motor speed controls |
| L298N Module | GND | GND | Black | Shared ground |
| Motor Encoder | OUT A (Signal) | GPIO4 | Yellow | Pulse interrupt pin |
| Motor Encoder | VCC / GND | 3V3 / GND | Red / Black | Encoder power rails |
| Potentiometer | Wiper | GPIO34 | Yellow | Target speed input |
| Potentiometer | VCC / GND | 3V3 / GND | Red / Black | Power rails |

> **Wiring tip:** Connect the encoder output to GPIO 4. In hardware, use a pull-up resistor (or internal pull-up in code) on the encoder output pin to ensure clean pulse transitions.

## Code
```cpp
// DC Motor PID Speed Controller (Encoder feedback)
#include <Arduino.h>

const int IN1 = 18;
const int IN2 = 19;
const int ENA_PIN = 5;
const int ENCODER_PIN = 4;
const int POT_PIN = 34;

// PWM settings
const int PWM_FREQ = 2000;
const int PWM_RES = 8;
const int LEDC_CHAN = 0;

// Encoder Variables
volatile int pulseCount = 0;
unsigned long lastRPMTime = 0;
const int PULSES_PER_REV = 20; // Slots on the encoder disk

// PID Variables
float kp = 1.8;
float ki = 0.5;
float kd = 0.1;

float targetRPM = 0.0;
float currentRPM = 0.0;
float error = 0.0;
float lastError = 0.0;
float integral = 0.0;
float derivative = 0.0;
float pidOutput = 0.0;

unsigned long lastPIDTime = 0;

// Interrupt Service Routine (ISR) called on every encoder pulse
void IRAM_ATTR countEncoderPulses() {
  pulseCount++;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  
  // Set motor forward direction
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  
  // Initialize ENA PWM pin using LEDC
  ledcSetup(LEDC_CHAN, PWM_FREQ, PWM_RES);
  ledcAttachPin(ENA_PIN, LEDC_CHAN);
  ledcWrite(LEDC_CHAN, 0); // Start stopped
  
  // Attach interrupt to encoder pin (rising edge)
  pinMode(ENCODER_PIN, INPUT_PULLUP);
  attachInterrupt(digitalPinToInterrupt(ENCODER_PIN), countEncoderPulses, RISING);
  
  lastRPMTime = millis();
  lastPIDTime = millis();
  
  Serial.println("PID Speed Controller online.");
}

void loop() {
  unsigned long now = millis();
  
  // 1. Calculate Current RPM every 100 ms
  if (now - lastRPMTime >= 100) {
    // Disable interrupts briefly to read pulse count safely
    noInterrupts();
    int pulses = pulseCount;
    pulseCount = 0;
    interrupts();
    
    // RPM = (pulses / PPR) * (60000ms / 100ms interval)
    currentRPM = ((float)pulses / (float)PULSES_PER_REV) * 600.0;
    lastRPMTime = now;
  }
  
  // 2. Run PID Loop every 50 ms
  if (now - lastPIDTime >= 50) {
    float dt = (now - lastPIDTime) / 1000.0; // Time interval in seconds
    lastPIDTime = now;
    
    // Read target RPM from potentiometer (scale 0-150 RPM)
    targetRPM = map(analogRead(POT_PIN), 0, 4095, 0, 150);
    
    // Calculate Error
    error = targetRPM - currentRPM;
    
    // Calculate Integral (with anti-windup clamping)
    integral += error * dt;
    integral = constrain(integral, -100, 100); 
    
    // Calculate Derivative
    derivative = (error - lastError) / dt;
    lastError = error;
    
    // Calculate PID output
    pidOutput = (kp * error) + (ki * integral) + (kd * derivative);
    
    // Scale PID output to PWM duty cycle (0-255)
    int pwmValue = constrain((int)pidOutput, 0, 255);
    
    // Command motor driver
    ledcWrite(LEDC_CHAN, pwmValue);
    
    // Log to serial monitor
    Serial.print("Target: "); Serial.print(targetRPM, 1);
    Serial.print(" | Current: "); Serial.print(currentRPM, 1);
    Serial.print(" | Error: "); Serial.print(error, 1);
    Serial.print(" | PWM: "); Serial.println(pwmValue);
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **L298N**, **DC Motor with Encoder**, and **Potentiometer** onto the canvas.
2. Wire ENA to **GPIO5**, IN1/IN2 to **GPIO18/GPIO19**, Encoder OUT to **GPIO4**, and Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer to set target RPM. Watch the motor spin and adjust its PWM to match the target.
5. Apply simulated friction load to the motor widget. Watch the PWM increase to compensate and keep RPM stable.

## Expected Output
Serial Monitor:
```
PID Speed Controller online.
Target: 60.0 | Current: 0.0 | Error: 60.0 | PWM: 108
Target: 60.0 | Current: 45.0 | Error: 15.0 | PWM: 125
Target: 60.0 | Current: 59.5 | Error: 0.5 | PWM: 130
```

## Expected Canvas Behavior
* Setting the potentiometer slider to 0 stops the motor widget.
* Setting the potentiometer to 60 makes the motor spin at approximately 60 RPM.
* Adding a friction load (by clicking the motor load options) causes the PWM to increase, keeping the wheel spinning at the same speed.

## Code Walkthrough
| Line | Math / Action |
| --- | --- |
| `IRAM_ATTR` | Directs the compiler to place the ISR function in the ESP32's fast Instruction RAM (IRAM) for rapid execution. |
| `attachInterrupt(...)` | Binds the encoder pin rising edges to trigger the pulse counter ISR. |
| `currentRPM = ...` | Computes revolutions per minute based on pulses counted over a 100 ms window. |
| `integral = constrain(...)` | Implements anti-windup clamping to prevent the integral term from accumulating to infinity if the motor is stalled. |

## Hardware & Safety Concept: Closed-Loop vs Open-Loop Control
In an **open-loop** system, the controller sends a fixed PWM value (e.g. 128) to the motor. If the robot goes uphill or carries a heavier load, friction slows the motor down, but the controller does not know. In a **closed-loop** system, the controller uses sensor feedback (encoder) to measure actual speed, compares it to the target speed, and uses a PID loop to adjust the PWM output, keeping the motor speed constant under variable loads.

## Try This! (Challenges)
1. **Interactive tuning console**: Allow the user to type new values for `kp`, `ki`, and `kd` in the Serial Monitor at runtime to test tuning behaviors.
2. **Dynamic overshoot logger**: Log the maximum speed overshoot (how much the speed spikes above target when starting) to test tuning stability.
3. **Stall Protection Cutoff**: If the PWM is at 255 but the RPM is 0 for more than 2 seconds, shut off the motor and trigger a warning to protect the coils.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor speed oscillates wildly | PID constants are too high | Reduce `kp` and `ki` constants by half |
| Motor starts spinning and slowly accelerates to max | Error sign or correction inverted | Ensure `error = target - current` is correct, and that a positive error increases the PWM output |
| RPM reads 0.0 | Encoder signal not received | Check encoder wiring; verify that the ISR is successfully incrementing the pulse counter |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [54 - ESP32 DC Motor Speed Scaling](../intermediate/54-esp32-dc-motor-speed-scaling.md)
- [123 - ESP32 Robot Speed Control](123-esp32-robot-speed-control.md)
- [152 - ESP32 Servo PID Angle Balancer](152-esp32-servo-pid-angle-balancer.md) (Next project)
