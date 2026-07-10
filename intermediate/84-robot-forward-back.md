# 84 - Robot Forward Back

Control a two-wheeled robot's direction (Forward and Backward) using an H-Bridge motor driver (L298N) and a switch.

## Goal
Learn how to use an H-bridge driver (L298N) to control the direction of DC motors by switching polarity, enabling forward and reverse motion.

## What You Will Build
The Arduino reads a toggle switch connected to pin D2.
- When the switch is HIGH, both motors spin forward (moving the robot straight).
- When the switch is LOW, both motors reverse directions (moving the robot backward).

**Why pins D3 to D8?** Pins D3 (ENA) and D8 (ENB) control the speed of the left and right wheels using PWM. Pins D4, D5, D6, and D7 are direction control lines (IN1, IN2, IN3, IN4).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| L298N Driver | `l298n_driver` | Yes | Yes |
| DC Motor | `dc_motor` | Yes (2 required) | Yes (2 required) |
| Slide Switch | `slide_switch` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Switch | Pin 1 (Middle) | D2 | Input signal line |
| Switch | Pin 2 | GND | Ground reference |
| L298N Driver | ENA | D3 | Left Motor Speed (PWM) |
| L298N Driver | IN1 | D4 | Left Motor Direction 1 |
| L298N Driver | IN2 | D5 | Left Motor Direction 2 |
| L298N Driver | IN3 | D6 | Right Motor Direction 1 |
| L298N Driver | IN4 | D7 | Right Motor Direction 2 |
| L298N Driver | ENB | D8 | Right Motor Speed (PWM) |
| L298N Driver | VIN | 5V | Power supply |
| L298N Driver | GND | GND | Ground reference |
| Left Motor | + / - | OUT1 / OUT2 | Driven by Left Channel |
| Right Motor | + / - | OUT3 / OUT4 | Driven by Right Channel |

## Code
```cpp
const int SWITCH_PIN = 2;

// Left Motor Control Pins
const int ENA = 3; 
const int IN1 = 4;
const int IN2 = 5;

// Right Motor Control Pins
const int IN3 = 6;
const int IN4 = 7;
const int ENB = 8;

void setup() {
  pinMode(SWITCH_PIN, INPUT_PULLUP);
  
  pinMode(ENA, OUTPUT);
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  pinMode(ENB, OUTPUT);
  
  Serial.begin(9600);
  Serial.println("L298N Robot Controller Ready");
}

void loop() {
  int directionSwitch = digitalRead(SWITCH_PIN);
  
  if (directionSwitch == HIGH) {
    Serial.println("Switch HIGH: Moving FORWARD");
    
    // Drive Left Motor Forward
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    analogWrite(ENA, 200); // 80% speed
    
    // Drive Right Motor Forward
    digitalWrite(IN3, HIGH);
    digitalWrite(IN4, LOW);
    analogWrite(ENB, 200); // 80% speed
  } 
  else {
    Serial.println("Switch LOW: Moving BACKWARD");
    
    // Drive Left Motor Backward
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, HIGH);
    analogWrite(ENA, 200);
    
    // Drive Right Motor Backward
    digitalWrite(IN3, LOW);
    digitalWrite(IN4, HIGH);
    analogWrite(ENB, 200);
  }
  
  delay(100); // Polling delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **L298N Driver**, **two DC Motors**, and **Slide Switch** onto the canvas.
2. Wire the driver inputs: **ENA** to **D3**, **IN1** to **D4**, **IN2** to **D5**, **IN3** to **D6**, **IN4** to **D7**, **ENB** to **D8**, **VIN** to **5V**, and **GND** to **GND**.
3. Wire the driver outputs: **OUT1/OUT2** to the first Motor, and **OUT3/OUT4** to the second Motor.
4. Connect Slide Switch middle pin to **D2** and an outer pin to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Click the Slide Switch on the canvas to toggle directions, and watch the motors spin forward and backward.

## Expected Output

Terminal:
```
L298N Robot Controller Ready
Switch HIGH: Moving FORWARD
Switch LOW: Moving BACKWARD
...
```

### Expected Canvas Behavior

| Switch State | Left Motor rotation | Right Motor rotation | Direction state |
| --- | --- | --- | --- |
| Open (HIGH) | Clockwise (Forward) | Clockwise (Forward) | Straight Ahead |
| Closed (LOW) | Counter-clockwise (Reverse) | Counter-clockwise (Reverse) | Straight Backwards |

Both motors change spin direction in sync when the switch is toggled.

## Code Walkthrough

| Pin / Constant | What It Does |
| --- | --- |
| `IN1` and `IN2` | Direction selection for Left Motor. If `IN1` is HIGH and `IN2` is LOW, current flows one way (forward). If reversed, current flows the other way (reverse). |
| `ENA` | Speed control for Left Motor. Outputs a PWM signal to regulate average voltage. |

## Hardware & Safety Concept: H-Bridges & High Currents
An **H-Bridge** is an electronic circuit containing four switches (usually MOSFET transistors) arranged in an "H" shape. By closing specific diagonal pairs of switches, the H-bridge changes the direction of the current flowing through the motor coil, reversing its rotation. 
- DC motors pull massive current when starting or stalling.
- The L298N module isolates the Arduino pins from these high currents and uses a separate battery input (typically 9V to 12V) to drive the motors.

## Try This! (Challenges)
1. **Pivot Turn**: Modify the code so that switching LOW makes the left motor spin forward and the right motor spin backward (causing the robot to spin in place).
2. **Speed Dial**: Wire a potentiometer to A0 and use it to regulate the speed of both motors dynamically (replacing the hardcoded `200` PWM value).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One motor spins forward, other spins backward | Motor terminal polarity reversed | Swap the (+) and (-) wires of the reversing motor at the L298N OUT terminal block. |
| Motors whine but don't spin | Speed PWM value too low | Low PWM values do not provide enough torque. Ensure your ENA/ENB output is set above 100. |

## Mode Notes
These patterns (multi-pin state selection and PWM driver calls) are supported by MbedO interpreted mode.

## Related Projects
- [82 - Motor Start Stop](82-motor-start-stop.md)
- [83 - Motor Speed Dial](83-motor-speed-dial.md)
- [85 - BT Remote Car](85-bt-remote-car.md)
