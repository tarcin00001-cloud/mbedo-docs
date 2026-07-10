# 20 - Pico Button OR Gate

Create a logical OR gate circuit using two push buttons and a warning LED.

## Goal
Learn how to combine multiple digital inputs using the logical OR operator (`||`) to trigger an output from either input source.

## What You Will Build
A dual-trigger alert console:
- **Button 1 (GP16)** & **Button 2 (GP17)**: Configured with internal pull-up resistors (active-LOW).
- **Warning LED (GP15)**: Turns ON if **either** button is pressed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Push Buttons | `button` | Yes (two buttons) | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Button 1 | Terminal 1 | GP16 | Sense pin 1 |
| Button 1 | Terminal 2 | GND | Ground return |
| Button 2 | Terminal 1 | GP17 | Sense pin 2 |
| Button 2 | Terminal 2 | GND | Ground return |
| Red LED | Anode | GP15 | Control output |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int BUTTON1 = 16;
const int BUTTON2 = 17;
const int LED_PIN = 15;

void setup() {
  pinMode(BUTTON1, INPUT_PULLUP);
  pinMode(BUTTON2, INPUT_PULLUP);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Active-LOW inputs: LOW means pressed
  int state1 = digitalRead(BUTTON1);
  int state2 = digitalRead(BUTTON2);

  // Logical OR check
  if (state1 == LOW || state2 == LOW) {
    digitalWrite(LED_PIN, HIGH); // Either pressed, light ON
  } else {
    digitalWrite(LED_PIN, LOW);  // Neither pressed, light OFF
  }

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Push Buttons**, and **Red LED** onto the canvas.
2. Connect Button 1: **Terminal 1** to **GP16**, **Terminal 2** to **GND**.
3. Connect Button 2: **Terminal 1** to **GP17**, **Terminal 2** to **GND**.
4. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
5. Paste code, select the interpreted mode, and click **Run**.
6. Click either push button to turn on the LED.

## Expected Output

Terminal:
```
Simulation active. GP16 and GP17 OR verification logic active.
```

## Expected Canvas Behavior
| Button 1 (GP16) | Button 2 (GP17) | LED state (GP15) |
| --- | --- | --- |
| open (Released) | open (Released) | LOW (OFF) |
| Closed (Pressed)| open (Released) | HIGH (ON) |
| open (Released) | Closed (Pressed)| HIGH (ON) |
| Closed (Pressed)| Closed (Pressed)| HIGH (ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `state1 == LOW \|\| state2 == LOW` | Evaluates true if at least one of the button inputs is pulled LOW to ground. |

## Hardware & Safety Concept: Redundant Alarms
OR gate logic is often used in safety override loops. A panic shutdown sequence can be triggered by hitting *either* the main stop button on the console *or* the remote stop switch mounted near the conveyor belt. 

## Try This! (Challenges)
1. **Alternate Indicator**: Make the LED blink slowly if Button 1 is pressed, and flash rapidly if Button 2 is pressed.
2. **Alert Tone**: Connect a buzzer to GP14 and sound it when either button is active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays ON when no buttons are pressed | Loose ground line | Confirm that the button terminal 2 wires are tied to a physical GND pin. |

## Mode Notes
This basic logical operation project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [16 - Pico Button LED](16-pico-button-led.md)
- [19 - Pico Button AND Gate](19-pico-button-and.md)
