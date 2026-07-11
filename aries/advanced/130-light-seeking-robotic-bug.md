# 130 - Light Seeking Robotic Bug

Program an autonomous robot to navigate towards light sources using dual Light Dependent Resistors (LDRs) and differential motor steering.

## Goal
Learn how to interface multiple analog sensors simultaneously, process analog differential values to infer direction, and implement light-seeking steering algorithms.

## What You Will Build
An autonomous light-seeking robotic bug. Two LDR (photoresistor) sensors are placed on the left and right front corners of the robot. The ARIES board reads the light intensity on both sides via analog inputs `ADC0` (GP26) and `ADC1` (GP27). By comparing the two light levels, the robot steers towards the brighter side. If it enters a completely dark room, it automatically stops to conserve power.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 2x LDR Photoresistors | `ldr` | Yes | Yes |
| 2x 10k-ohm Resistors | `resistor` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| 2x DC Toy Motors | `dc_motor` | Yes | Yes |
| External 6V-12V Power Source | `power_supply` | Optional | Yes (battery pack) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Left LDR | Pin 1 / Pin 2 | 3V3 / ADC0 (GP26) | Red / White | Left light sensor (voltage divider) |
| Left Resistor (10k) | Pin 1 / Pin 2 | ADC0 (GP26) / GND | White / Black | Pull-down resistor |
| Right LDR | Pin 1 / Pin 2 | 3V3 / ADC1 (GP27) | Red / White | Right light sensor (voltage divider) |
| Right Resistor (10k) | Pin 1 / Pin 2 | ADC1 (GP27) / GND | White / Black | Pull-down resistor |
| L298N Module | VCC / GND | 5V / GND | Red / Black | Driver power connections |
| L298N Module | IN1 / IN2 | GPIO 14 / 15 | Orange / Brown | Left motor control |
| L298N Module | IN3 / IN4 | GPIO 13 / 12 | Green / Blue | Right motor control |

> **Wiring tip:** Build a voltage divider for each LDR: Connect one leg of the LDR to 3.3V. Connect the other leg of the LDR to the ARIES analog pin (GP26 or GP27) and also to a 10k-ohm resistor that connects to GND. This pulls the analog input voltage up towards 3.3V as light intensity increases (higher light = higher ADC value).

## Code
```cpp
// Light Seeking Robotic Bug - VEGA ARIES v3
const int LDR_LEFT = GP26;  // Left LDR on ADC0
const int LDR_RIGHT = GP27; // Right LDR on ADC1

const int IN1 = 14;         // Left Motor Forward
const int IN2 = 15;         // Left Motor Backward
const int IN3 = 13;         // Right Motor Forward
const int IN4 = 12;         // Right Motor Backward

// Configuration constants
const int LIGHT_THRESHOLD = 500;  // Minimum ambient light to start moving
const int STEER_THRESHOLD = 250;  // Minimum difference to trigger steering (dead-band)

void setup() {
  // Motor driver pins set as outputs
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);

  // Initialize Serial Monitor
  Serial.begin(9600);
  Serial.println("Light Seeking Bug Initialized.");
}

void loop() {
  // Read analog values from both LDR sensors (12-bit ADC: 0 - 4095)
  int leftVal = analogRead(LDR_LEFT);
  int rightVal = analogRead(LDR_RIGHT);

  Serial.print("LDR Left: ");
  Serial.print(leftVal);
  Serial.print(" | LDR Right: ");
  Serial.println(rightVal);

  // Check if both sides are dark
  if (leftVal < LIGHT_THRESHOLD && rightVal < LIGHT_THRESHOLD) {
    // Too dark: Stop the robot
    Serial.println("State: DARK - Stopped");
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, LOW);
  } 
  else {
    // Compare light levels
    int diff = leftVal - rightVal;

    if (diff > STEER_THRESHOLD) {
      // Left side is significantly brighter -> steer left (stop left wheel, run right wheel)
      Serial.println("State: Light on Left - Steering Left");
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
    } 
    else if (diff < -STEER_THRESHOLD) {
      // Right side is significantly brighter -> steer right (run left wheel, stop right wheel)
      Serial.println("State: Light on Right - Steering Right");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, LOW);
      digitalWrite(IN4, LOW);
    } 
    else {
      // Light is roughly balanced -> go straight forward
      Serial.println("State: Balanced - Driving Forward");
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      digitalWrite(IN3, HIGH);
      digitalWrite(IN4, LOW);
    }
  }

  delay(100); // 10Hz sampling rate
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, two **DC Toy Motors**, and two **LDRs** onto the canvas.
2. Wire the L298N module power and outputs to the DC motors.
3. Wire L298N inputs: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **IN3** to **GPIO 13**, and **IN4** to **GPIO 12**.
4. Set up the voltage dividers: Connect Left LDR to **GP26 (ADC0)** and Right LDR to **GP27 (ADC1)**.
5. Paste the C++ code into the editor.
6. Select **Interpreted Mode** and click **Run**.
7. Adjust the light level sliders on both simulated LDRs.
8. Observe the motor behaviors: if you slide the left LDR to 3500 and the right to 1200, the left motor stops and the right motor spins to turn the robot towards the light.

## Expected Output
Serial Monitor:
```
Light Seeking Bug Initialized.
LDR Left: 320 | LDR Right: 280
State: DARK - Stopped
LDR Left: 2400 | LDR Right: 2350
State: Balanced - Driving Forward
LDR Left: 3100 | LDR Right: 1200
State: Light on Left - Steering Left
```

## Expected Canvas Behavior
* Both LDR sliders < 500: Both motors remain stopped.
* Left LDR slider > Right LDR slider by 250+: The left motor stops, and the right motor spins forward.
* Right LDR slider > Left LDR slider by 250+: The right motor stops, and the left motor spins forward.
* Both LDR sliders high and close: Both motors spin forward.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(LDR_LEFT)` | Reads the 12-bit voltage value representing the light level on the left side (0 to 4095). |
| `leftVal < LIGHT_THRESHOLD ...` | Checks if both sensors are in low-light conditions to execute power-saving stops. |
| `leftVal - rightVal` | Computes the light level differential to determine the brighter side. |
| `diff > STEER_THRESHOLD` | Evaluates if the difference exceeds the dead-band limit, filtering out minor fluctuations. |

## Hardware & Safety Concept
* **Voltage Dividers with Analog Inputs**: Analog input pins on microcontrollers measure voltage, not raw resistance. To read a changing resistance from an LDR, we must pair it with a fixed resistor (e.g. 10k-ohm) to form a voltage divider. This configuration scales the sensor's changing resistance into a voltage curve between 0V and 3.3V.
* **Ambient Calibration and Phototaxis**: In phototaxis robotics, shadows or varying wall reflections can confuse the robot. Placing short, dark plastic tubes (light shields or "blinders") around each photoresistor narrows their viewing angles. This directional shielding ensures the LDRs only detect light coming directly from their respective sides, improving tracking precision.

## Try This! (Challenges)
1. **Light Avoiding Bug (Photophobic)**: Modify the motor direction steering logic so the robot turns *away* from light sources and seeks out dark spots (such as driving under tables or beds).
2. **Speed Scaling Seeking**: Integrate PWM control on LENA/ENB. Make the robot drive faster when the light source is far away (large differences) and slow down as it gets closer and the light balances out.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The robot keeps turning away from the light | Left and right LDR code variables are swapped | Swap the input pin assignments (`LDR_LEFT` and `LDR_RIGHT`) in the code, or swap the physical pins. |
| The robot is jittering/switching back and forth constantly | Steering threshold (`STEER_THRESHOLD`) is too small | Increase `STEER_THRESHOLD` to 400 or 500 to widen the steering dead-band and stabilize travel. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - LDR Automatic Dark Detector](../beginner/34-ldr-automatic-dark-detector.md)
- [48 - LDR Ambient Light LED Dimmer](../beginner/48-ldr-ambient-light-led-dimmer.md)
