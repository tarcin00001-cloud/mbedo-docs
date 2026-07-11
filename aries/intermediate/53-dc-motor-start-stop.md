# 53 - DC Motor Start/Stop (via L298N)

Start and stop a DC motor at regular intervals using an L298N H-bridge motor driver and a loop-free C++ state machine.

## Goal
Learn how to interface the VEGA ARIES v3 board with an L298N motor driver to control the power and state of a DC motor without using loops inside C++ code.

## What You Will Build
A DC motor controller that turns the motor ON for 3 seconds, then stops it for 3 seconds, repeating the cycle in a loop-free C++ state machine.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 5V DC Motor | `motor` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | IN1 | GPIO 14 | Blue | Control input 1 |
| L298N Driver | IN2 | GPIO 15 | Yellow | Control input 2 |
| L298N Driver | ENA | 5V | Red | Enable jumper active |
| L298N Driver | VCC | 5V | Red | Power input |
| L298N Driver | GND | GND | Black | Ground reference |
| DC Motor | Terminal 1 | OUT1 | Red | Driver output 1 |
| DC Motor | Terminal 2 | OUT2 | Black | Driver output 2 |

> **Wiring tip:** The L298N ENA (Enable A) pin must be connected to 5V (either via a physical jumper on the driver board or a jumper wire) to enable the H-bridge channel at full speed.

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;
int motorState = 0; // 0 = stopped, 1 = running forward

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
}

void loop() {
  if (motorState == 0) {
    // Turn motor ON (Forward)
    digitalWrite(IN1, HIGH);
    digitalWrite(IN2, LOW);
    motorState = 1;
  } else {
    // Turn motor OFF (Stop)
    digitalWrite(IN1, LOW);
    digitalWrite(IN2, LOW);
    motorState = 0;
  }
  delay(3000); // Keep current state active for 3 seconds
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, and **DC Motor** components onto the canvas.
2. Connect L298N: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **ENA** to **5V**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the DC Motor terminals to **OUT1** and **OUT2** of the L298N.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Motor Running.
Motor Stopped.
```

## Expected Canvas Behavior
* The DC motor spins in one direction for 3 seconds, stops completely for 3 seconds, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(IN1, OUTPUT)` | Configures control pin 1 as a digital output. |
| `digitalWrite(IN1, HIGH)` | Drives IN1 to HIGH, routing power through H-Bridge OUT1. |
| `digitalWrite(IN2, LOW)` | Drives IN2 to LOW, routing H-Bridge OUT2 to ground. |
| `motorState = 1` | Updates state variable to indicate motor is running. |
| `delay(3000)` | Keeps the current state for 3 seconds. |

## Hardware & Safety Concept
* **H-Bridge Driver**: Microcontrollers cannot directly drive DC motors because motors draw too much current and generate reverse electromotive force (Back EMF). The L298N H-bridge uses high-current transistors to switch power to the motor based on low-current control signals from the ARIES board.
* **Inductive Kickback**: Always use a motor driver with flyback diodes (like the L298N module) to suppress voltage spikes from the motor coils when turning them off, preventing damage to the silicon.

## Try This! (Challenges)
1. **Direction Reverse**: Modify the code to run forward for 2 seconds, stop for 1 second, run backward for 2 seconds, and stop for 1 second.
2. **Shorten Cycle**: Change the delays to 1 second to create a rapid pulsing start-stop behavior.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not rotate | ENA is not connected or jumper is missing | Make sure the ENA pin on the L298N has its jumper installed or is connected to 5V. |
| Motor hums but doesn't spin | Low voltage or current limit | Connect an external battery to the L298N's power terminals if the board USB port cannot supply enough current. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [10 - Relay Module Switch](../beginner/10-relay-module-switch.md)
- [54 - DC Motor Speed Scaling (PWM ENA)](54-dc-motor-speed-scaling.md)
