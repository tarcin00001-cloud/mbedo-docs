# 55 - DC Motor Direction Toggle

Change the rotation direction of a DC motor using an external push button and an L298N motor driver with a loop-free C++ state machine.

## Goal
Learn how to read button states using internal pull-up resistors on the VEGA ARIES v3 board and use them to toggle the direction pins of an H-bridge driver without using loops inside C++ code.

## What You Will Build
A motor control system where pressing a push button connected to GPIO 16 reverses the rotation direction of a DC motor driven by an L298N module.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 5V DC Motor | `motor` | Yes | Yes |
| Push Button | `button` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | IN1 | GPIO 14 | Blue | Control input 1 |
| L298N Driver | IN2 | GPIO 15 | Yellow | Control input 2 |
| L298N Driver | ENA | 5V | Red | Enable H-bridge |
| L298N Driver | VCC | 5V | Red | Power input |
| L298N Driver | GND | GND | Black | Ground reference |
| Push Button | Pin 1 | GPIO 16 | Green | Button signal pin |
| Push Button | Pin 2 | GND | Black | Ground return |
| DC Motor | Terminal 1 | OUT1 | Red | Driver output 1 |
| DC Motor | Terminal 2 | OUT2 | Black | Driver output 2 |

> **Wiring tip:** The push button is connected between GPIO 16 and GND. Enabling the internal pull-up resistor on GPIO 16 ensures the pin reads HIGH when the button is open, and LOW when pressed.

## Code
```cpp
const int IN1 = 14;
const int IN2 = 15;
const int BUTTON_PIN = 16;
int lastButtonState = HIGH;
int motorDirection = 0; // 0 = forward, 1 = reverse

void setup() {
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(BUTTON_PIN, INPUT_PULLUP);

  // Start with motor running forward
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
}

void loop() {
  int currentButtonState = digitalRead(BUTTON_PIN);

  // Detect button press transition (falling edge: HIGH to LOW)
  if (currentButtonState == LOW && lastButtonState == HIGH) {
    if (motorDirection == 0) {
      // Toggle to Reverse
      digitalWrite(IN1, LOW);
      digitalWrite(IN2, HIGH);
      motorDirection = 1;
    } else {
      // Toggle to Forward
      digitalWrite(IN1, HIGH);
      digitalWrite(IN2, LOW);
      motorDirection = 0;
    }
    delay(200); // Mechanical contact debounce delay
  }

  lastButtonState = currentButtonState;
  delay(10); // Short polling interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **L298N Motor Driver**, **DC Motor**, and **Push Button** components onto the canvas.
2. Connect L298N: **IN1** to **GPIO 14**, **IN2** to **GPIO 15**, **ENA** to **5V**, **VCC** to **5V**, and **GND** to **GND**.
3. Connect the Push Button: **Pin 1** to **GPIO 16**, **Pin 2** to **GND**.
4. Connect the DC Motor to the L298N outputs **OUT1** and **OUT2**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Direction: FORWARD
Direction: REVERSE
```

## Expected Canvas Behavior
* The DC motor spins in the forward direction. When the push button on GPIO 16 is clicked, the motor reverses its rotation direction. Clicking the button again swaps it back.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUTTON_PIN, INPUT_PULLUP)` | Configures GPIO 16 as input with internal pull-up resistor. |
| `digitalRead(BUTTON_PIN)` | Reads the current state of the button (LOW = pressed). |
| `currentButtonState == LOW && lastButtonState == HIGH` | State edge detection logic to identify a fresh button press. |
| `digitalWrite(IN1, LOW); digitalWrite(IN2, HIGH);` | Reverses polarity on L298N OUT1/OUT2 pins to reverse current. |
| `delay(200)` | Prevents bouncing contacts from triggering multiple state changes. |

## Hardware & Safety Concept
* **H-Bridge Polarity Swap**: Direct current motors rotate based on current flow direction. By swapping the high/low state of IN1 and IN2, the H-Bridge flips the direction of the voltage applied across OUT1 and OUT2, reversing the magnetic field and direction of rotation.
* **Back EMF Suppression**: Rapidly reversing direction while the motor is spinning at high speed can cause high Back EMF spikes and stress the H-Bridge. For physical motors, it is safer to stop the motor (IN1=LOW, IN2=LOW) for a brief duration before changing direction.

## Try This! (Challenges)
1. **Three-State Controller**: Modify the code to cycle through three states: Forward, Stop, Reverse, Stop on successive button presses.
2. **Onboard Color Indicators**: Configure the onboard RGB LED to turn GREEN during forward rotation, RED during reverse rotation, and OFF when stopped.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor direction does not swap when button is clicked | Button is not wired to GND | Ensure one terminal of the push button connects to GND and the other to GPIO 16. |
| Motor swaps direction continuously | Missing debounce or wrong pin state | Ensure `INPUT_PULLUP` is set in setup and `delay(200)` debounce is active. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [53 - DC Motor Start/Stop (via L298N)](53-dc-motor-start-stop.md)
- [56 - Stepper Motor Step Sequence (ULN2003)](56-stepper-motor-step-sequence.md)
