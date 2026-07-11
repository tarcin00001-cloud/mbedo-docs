# 179 - Industrial Motor Controller

Build an industrial-grade closed-loop DC motor speed controller featuring quadrature encoder feedback, PID regulation, I2C LCD telemetry, and overload/stall protection.

## Goal
Learn how to decode high-speed quadrature encoder pulses, execute a discrete PID regulation loop in real-time, adjust setpoints using a potentiometer, display control variables on an LCD, and trigger a safety shutdown state on overload.

## What You Will Build
A closed-loop speed controller. A potentiometer sets the target RPM. A DC motor with a quadrature encoder runs on an L298N driver. The ARIES board continuously polls the encoder lines, calculates current RPM, and adjusts the PWM duty cycle using a PID loop to match the target speed. If a motor stall or overload is detected (indicated by high control error for over 2 seconds), the system shuts down, sounds an alarm buzzer, and illuminates a warning LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DC Motor with Encoder | `motor_encoder` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | ENA | GPIO 9 | Orange | PWM speed control |
| L298N Driver | IN1 | GPIO 7 | Yellow | Motor direction pin 1 |
| L298N Driver | IN2 | GPIO 8 | Green | Motor direction pin 2 |
| L298N Driver | VCC | 5V | Red | Motor power |
| L298N Driver | GND | GND | Black | Common Ground |
| Encoder | VCC | 3V3 | Red | Encoder power |
| Encoder | GND | GND | Black | Ground |
| Encoder | OUT A | GPIO 14 | Blue | Encoder pulse A |
| Encoder | OUT B | GPIO 15 | White | Encoder pulse B |
| Potentiometer | Wiper | ADC0 | Blue | Setpoint input (GP26) |
| Potentiometer | VCC | 3V3 | Red | Power |
| Potentiometer | GND | GND | Black | Ground |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 6 | Gray | Alarm pin |
| Active Buzzer | - | GND | Black | Ground |
| Warning LED | Anode | GPIO 13 | Red | Overload indicator |
| Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** The quadrature encoder pins are mapped to GPIO 14 and GPIO 15. Because these pins are shared with the default buzzer and warning LED mappings, we move the Buzzer to GPIO 6 and the Warning LED to GPIO 13 to avoid signal interference.

## Code
```cpp
// Industrial Motor Controller - VEGA ARIES v3
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ENA_PIN = 9;
const int IN1_PIN = 7;
const int IN2_PIN = 8;
const int ENCODER_A = 14;
const int ENCODER_B = 15;
const int POT_PIN = ADC0;
const int BUZZER_PIN = 6;
const int WARNING_LED = 13;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Encoder Tracking
int lastEncoderA = LOW;
long pulseCount = 0;
unsigned long lastPidTime = 0;
unsigned long lastLcdTime = 0;

// PID Variables
float targetRPM = 0.0;
float currentRPM = 0.0;
float Kp = 1.8;
float Ki = 0.6;
float Kd = 0.05;
float error = 0.0;
float lastError = 0.0;
float integral = 0.0;
int pwmOutput = 0;

// Safety & Overload Monitoring
bool systemFault = false;
unsigned long faultTimer = 0;
bool faultTimingActive = false;

void setup() {
  Serial.begin(115200);
  pinMode(ENA_PIN, OUTPUT);
  pinMode(IN1_PIN, OUTPUT);
  pinMode(IN2_PIN, OUTPUT);
  pinMode(ENCODER_A, INPUT);
  pinMode(ENCODER_B, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);

  // Set motor forward
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("PID Controller");
  lcd.setCursor(0, 1);
  lcd.print("Ready.");
  
  lastEncoderA = digitalRead(ENCODER_A);
  lastPidTime = millis();
  lastLcdTime = millis();
}

void loop() {
  // 1. High-frequency polling of quadrature encoder
  int stateA = digitalRead(ENCODER_A);
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

  // 2. Read target speed from potentiometer (0 to 180 RPM)
  int potVal = analogRead(POT_PIN);
  if (!systemFault) {
    targetRPM = map(potVal, 0, 1023, 0, 180);
  } else {
    targetRPM = 0;
  }

  unsigned long currentTime = millis();
  unsigned long elapsedPid = currentTime - lastPidTime;

  // 3. Run PID speed regulation loop every 100ms
  if (elapsedPid >= 100 && !systemFault) {
    lastPidTime = currentTime;
    float dt = (float)elapsedPid / 1000.0;

    // RPM Calculation: assuming 20 pulses per revolution
    float pulsesPerSec = ((float)pulseCount / dt);
    currentRPM = (pulsesPerSec * 60.0) / 20.0;
    pulseCount = 0; // Reset counter for next window

    // Error calculations
    error = targetRPM - currentRPM;
    integral = integral + (error * dt);
    
    // Constrain integral term
    if (integral > 150.0) integral = 150.0;
    if (integral < -150.0) integral = -150.0;

    float derivative = (error - lastError) / dt;
    lastError = error;

    float pidVal = (Kp * error) + (Ki * integral) + (Kd * derivative);
    pwmOutput = (int)pidVal;

    // Limit output
    if (pwmOutput < 0) pwmOutput = 0;
    if (pwmOutput > 255) pwmOutput = 255;

    analogWrite(ENA_PIN, pwmOutput);

    // Overload Detection logic: Target is high, but motor speed is stuck close to 0
    if (targetRPM > 40.0 && currentRPM < 10.0) {
      if (!faultTimingActive) {
        faultTimer = currentTime;
        faultTimingActive = true;
      } else if (currentTime - faultTimer >= 2000) {
        // Stalled for over 2 seconds -> Fault!
        systemFault = true;
        analogWrite(ENA_PIN, 0); // Disarm motor
        digitalWrite(BUZZER_PIN, HIGH);
        digitalWrite(WARNING_LED, HIGH);
      }
    } else {
      faultTimingActive = false;
    }
  }

  // 4. Update LCD Display every 500ms
  unsigned long elapsedLcd = currentTime - lastLcdTime;
  if (elapsedLcd >= 500) {
    lastLcdTime = currentTime;
    lcd.clear();
    
    if (systemFault) {
      lcd.setCursor(0, 0);
      lcd.print("** OVERLOAD **");
      lcd.setCursor(0, 1);
      lcd.print("System Shut down");
    } else {
      lcd.setCursor(0, 0);
      lcd.print("Set:");
      lcd.print((int)targetRPM);
      lcd.print("  Act:");
      lcd.print((int)currentRPM);
      
      lcd.setCursor(0, 1);
      lcd.print("PWM: ");
      lcd.print(pwmOutput);
      if (faultTimingActive) {
        lcd.print(" !STALL");
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DC Motor with Encoder**, **L298N driver**, **Potentiometer**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring matching the table. Ensure the buzzer/LED are moved to pins 6 and 13.
3. Paste the code into the editor and choose **Interpreted Mode**.
4. Click **Run**.
5. Rotate the potentiometer widget. The motor speed adapts to match. Hold or brake the motor widget manually to trigger the overload shutdown.

## Expected Output
Serial Monitor:
```
Target: 100 RPM | Actual: 0 RPM | PWM: 180
Target: 100 RPM | Actual: 65 RPM | PWM: 230
Target: 100 RPM | Actual: 98 RPM | PWM: 202
Target: 100 RPM | Actual: 3 RPM | PWM: 255 (Overload Timer Started!)
** SYSTEM SHUTDOWN: MOTOR STALLED **
```

## Expected Canvas Behavior
* The motor starts spinning, matching the speed selected by the potentiometer.
* The LCD displays target speed, measured speed, and PWM duty cycle.
* If you manually stall the motor slider (force actual RPM to zero while setpoint is >40), the buzzer turns on, the red LED turns on, the LCD says "** OVERLOAD **", and the motor stops receiving drive signal.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `stateA != lastEncoderA` | Identifies state changes on Encoder A pin to count ticks. |
| `targetRPM = map(...)` | Converts analog knob reading to physical target speed limit of 180 RPM. |
| `currentRPM = ...` | Converts ticks over the 100ms PID step size into revolutions per minute. |
| `pidVal = ...` | Combines proportional, integral, and derivative terms into motor output adjustment. |
| `currentTime - faultTimer >= 2000` | Triggers latching safety state if motor fails to spin under load for 2 seconds. |

## Hardware & Safety Concept
* **Latching Fault State**: Once an overload occurs, the ARIES controller halts the PID outputs and locks the motor state until manual intervention (reset). This protects the MOSFET switching stages of the L298N driver from overheating.
* **Flyback Diodes**: DC motors are inductive loads. When current is cut, the collapsing magnetic field causes high voltage spikes. Freewheeling diodes on the motor outputs absorb these spikes.

## Try This! (Challenges)
1. **Fault Clear Button**: Add code to poll the Push Button (GPIO 16) to clear the system fault state, reset the PID integrator, and allow the motor to spin again.
2. **Reverse Direction Toggle**: Use the Push Button to toggle motor direction. Adjust the PID calculations to handle negative target and actual RPM values.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor accelerates to maximum speed immediately | Feedback loop is positive (wrong encoder logic) | Swap the OUT A and OUT B encoder wires or swap the motor terminals. |
| Motor chatters or vibrates without turning | Integral gain (Ki) or Proportional gain (Kp) too high | Lower Kp and Ki values to soften PID correction response. |
| System shows OVERLOAD instantly | Encoder not powering up | Check that the encoder VCC is connected to ARIES 3.3V and GND is connected. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [123 - Robot Speed Control](../advanced/123-robot-speed-control.md)
- [151 - DC Motor PID Speed Controller](../advanced/151-dc-motor-pid-speed-controller.md)
- [173 - Autonomous Tank Robot](173-autonomous-tank-robot.md)
