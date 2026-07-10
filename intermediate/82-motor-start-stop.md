# 82 - Motor Start Stop

Start and stop a DC motor using a pushbutton toggle switch.

## Goal
Learn how to use a digital input button to toggle a higher-current motor load ON and OFF using software state latching.

## What You Will Build
Each time the button on D2 is pressed, the DC motor connected to pin D9 toggles its operating state: if it was stopped, it starts spinning at full speed; if it was spinning, it stops.

**Why D2 and D9?** Pin D2 reads the push button input. Pin D9 outputs the control signal to switch the motor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| NPN Transistor (e.g. PN2222) | `transistor` | Optional | Yes (critical driver) |
| Flyback Diode (e.g. 1N4007) | `diode` | Optional | Yes (critical protector) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Push Button | Pin 1 | D2 | Input control signal |
| Push Button | Pin 2 | GND | Ground reference |
| DC Motor | + | D9 | Output switch control |
| DC Motor | - | GND | Ground reference |

> [!IMPORTANT]
> **Hardware Safety Warning:** A physical DC motor must **never** be connected directly to an Arduino output pin, as it draws too much current and will damage the board. You must use a transistor (like a PN2222) as a driver, along with a flyback diode.

## Code
```cpp
const int BTN_PIN   = 2;
const int MOTOR_PIN = 9;

int motorState = LOW;     // Current state of the motor (LOW = off, HIGH = on)
int lastBtnState = HIGH;  // Previous reading from the input pin

void setup() {
  pinMode(BTN_PIN, INPUT_PULLUP);
  pinMode(MOTOR_PIN, OUTPUT);
  digitalWrite(MOTOR_PIN, motorState); // Initialize motor off
  
  Serial.begin(9600);
  Serial.println("Motor Toggle Latch Ready");
}

void loop() {
  int btnState = digitalRead(BTN_PIN);
  
  // Detect button press (falling edge: was HIGH, is now LOW)
  if (btnState == LOW && lastBtnState == HIGH) {
    // Toggle the state
    if (motorState == LOW) {
      motorState = HIGH;
      Serial.println("[ACTION] Motor Started");
    } else {
      motorState = LOW;
      Serial.println("[ACTION] Motor Stopped");
    }
    
    digitalWrite(MOTOR_PIN, motorState);
    
    delay(150); // Debounce delay
  }
  
  lastBtnState = btnState;
  delay(10);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Push Button**, and **DC Motor** onto the canvas.
2. Connect Button: **pin 1** to Arduino **D2** and **pin 2** to Arduino **GND**.
3. Connect DC Motor: **+** to Arduino **D9** and **-** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the Push Button on the canvas, and watch the DC Motor shaft spin and stop.

## Expected Output

Terminal:
```
Motor Toggle Latch Ready
[ACTION] Motor Started
[ACTION] Motor Stopped
...
```

### Expected Canvas Behavior

| Button Press Action | Latch State comparison | Pin D9 State | Motor Shaft |
| --- | --- | --- | --- |
| Released | No change | Keeps previous | No change |
| Click 1 | Off -> On | HIGH (5V) | Spins at Max |
| Click 2 | On -> Off | LOW (0V) | Stopped |

The motor toggles state instantly upon register of a click.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `motorState = (motorState == LOW) ? HIGH : LOW;` | Simplified toggle logic (expanded in the code for readability). Swaps state between HIGH and LOW. |
| `digitalWrite(MOTOR_PIN, motorState)` | Updates the output pin voltage to match the latched state. |

## Hardware & Safety Concept: Latching States
In software design, **state latching** is the practice of converting a momentary input event (like pushing a button) into a persistent output state (like keeping a motor running after the button is released). This is the electrical equivalent of a clicky push-button switch.

## Try This! (Challenges)
1. **Pulse Run**: Modify the code so that the motor only spins *while* you hold the button down, and stops immediately when you release it.
2. **Speed Cycle**: Instead of ON/OFF, make the button cycle through three speeds: OFF -> Medium (120 PWM) -> Fast (255 PWM) -> OFF. (Hint: change pin D9 drive using `analogWrite()`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor vibrates but won't start spinning | Button bounce or low current | Verify that the button debounce delay of `150` ms is present. |
| Motor spins constantly without toggle | Pin mode error | Ensure input pin is set to `INPUT_PULLUP` to keep input line stable when released. |

## Mode Notes
These patterns (digital input edge detection driving output toggles) are supported by MbedO interpreted mode.

## Related Projects
- [06 - Button Edge Detection](../beginner/06-button-edge-detection.md)
- [37 - Night Fan Relay](37-night-fan-relay.md)
- [83 - Motor Speed Dial](83-motor-speed-dial.md)
