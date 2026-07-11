# 54 - DC Motor Speed Scaling (PWM ENA)

Control a DC motor's rotation speed by applying a Pulse Width Modulation (PWM) signal to the enable pin of the L298N driver using a loop-free C++ state machine.

## Goal
Learn how to use `analogWrite()` to output PWM signals from the VEGA ARIES v3 board to scale the speed of a DC motor smoothly without using loops inside C++ code.

## What You Will Build
A speed control system where the DC motor speed fades up from 0 to 255 (maximum) and then fades down to 0, repeating continuously in a loop-free manner.

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
| L298N Driver | ENA | GPIO 13 | Orange | PWM speed control line |
| L298N Driver | VCC | 5V | Red | Power input |
| L298N Driver | GND | GND | Black | Ground reference |
| DC Motor | Terminal 1 | OUT1 | Red | Driver output 1 |
| DC Motor | Terminal 2 | OUT2 | Black | Driver output 2 |

> **Wiring tip:** Remove the physical jumper cap from the L298N ENA pin header. Connect ENA directly to GPIO 13 on the ARIES board to allow PWM signal control.

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;
const int ENA = 13;
int speed = 0;
int speedStep = 15;

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(ENA, OUTPUT);

  // Set initial motor direction to forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
}

void loop() {
  analogWrite(ENA, speed); // Output PWM speed signal to ENA pin
  speed += speedStep;

  // Reverse speed fading direction at endpoints (0 and 255)
  if (speed >= 255 || speed <= 0) {
    speedStep = -speedStep;
  }

  delay(150); // Pause to see the motor speed scale
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, and **DC Motor** components onto the canvas.
2. Connect L298N: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **ENA** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the DC Motor to the L298N outputs **OUT1** and **OUT2**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
PWM Speed set: 0
PWM Speed set: 45
PWM Speed set: 120
PWM Speed set: 255
```

## Expected Canvas Behavior
* The DC motor starts spinning slowly, increases speed until it spins at maximum, then slows down to a stop, and repeats the cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(ENA, OUTPUT)` | Configures the ENA pin as output. |
| `digitalWrite(IN1, HIGH)` | Configures the H-bridge direction for forward rotation. |
| `analogWrite(ENA, speed)` | Writes a PWM duty cycle (0–255) to the speed control pin. |
| `speed += speedStep` | Modifies the speed value loop-free on each iteration. |
| `speedStep = -speedStep` | Reverses the speed fading direction at maximum (255) or minimum (0) limits. |

## Hardware & Safety Concept
* **PWM Duty Cycle**: PWM controls motor speed by turning the power ON and OFF extremely fast. The ratio of ON time to the total cycle time (duty cycle) determines the average voltage seen by the motor: 0 is 0% voltage, 128 is 50% voltage, and 255 is 100% voltage.
* **Switching Losses**: High PWM frequencies can cause thermal buildup in driver transistors due to transition switching losses. Standard microcontroller PWM defaults (around 490 Hz or 1 kHz) provide a good balance between motor response and driver heating.

## Try This! (Challenges)
1. **Direction Swap Fading**: Modify the code to fade the speed from 0 to 255 in the forward direction, stop, switch direction, and fade from 0 to 255 in reverse.
2. **Stepped Output**: Change the fade step to `50` and the delay to `300` to create distinct speed steps instead of a smooth fade.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor spins only at maximum speed | ENA jumper still connected | Make sure you have physically removed the black jumper on the ENA pins of the L298N module and connected ENA to GPIO 13. |
| Motor hums at low PWM values but does not rotate | High static friction | The motor needs a minimum voltage (e.g., PWM > 60) to overcome initial friction. Start the fade at a higher base speed or map values. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - DC Motor Start/Stop (via L298N)](53-dc-motor-start-stop.md)
- [55 - DC Motor Direction Toggle](55-dc-motor-direction-toggle.md)
