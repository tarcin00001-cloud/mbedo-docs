# 22 - Pico Touch Toggle

Build a toggle-switch touch lamp utilizing a TTP223 capacitive touch sensor and an LED.

## Goal
Learn how to implement a state-latching toggle switch in code using momentary sensor inputs.

## What You Will Build
A latching touch lamp:
- **TTP223 Touch Sensor (GP16)**: Pressing and releasing the touch pad toggles the lamp state.
- **Warning LED (GP15)**: Switches ON after a touch, and remains ON until the pad is touched a second time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| TTP223 Touch Sensor | `ttp223` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Touch Sensor | VCC | 3.3V | Power supply |
| Touch Sensor | I/O | GP16 | Touch signal |
| Touch Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Lamp control |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int TOUCH_PIN = 16;
const int LED_PIN   = 15;

bool lampState = false;
bool lastTouchState = false;

void setup() {
  pinMode(TOUCH_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with lamp OFF
}

void loop() {
  int currentTouchState = digitalRead(TOUCH_PIN);

  // State-change detection: touch transitioned from NOT touched to touched (rising edge)
  if (currentTouchState == HIGH && lastTouchState == LOW) {
    lampState = !lampState; // Invert lamp state
    digitalWrite(LED_PIN, lampState ? HIGH : LOW);
    delay(200); // Debounce delay to prevent double triggers
  }

  lastTouchState = currentTouchState;
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **TTP223 Touch Sensor**, and **Red LED** onto the canvas.
2. Connect Touch Sensor: **VCC** to **3.3V**, **I/O** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Click the touch sensor once to turn the LED ON, and click it again to turn it OFF.

## Expected Output

Terminal:
```
Simulation active. Toggle switch logic ready.
```

## Expected Canvas Behavior
| Action | Touch Input (GP16) | LED state (GP15) | Lamp State |
| --- | --- | --- | --- |
| Initial state | LOW | LOW | OFF |
| First click (Touch) | HIGH | HIGH | ON |
| Release touch | LOW | HIGH | ON |
| Second click (Touch) | HIGH | LOW | OFF |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `currentTouchState == HIGH && lastTouchState == LOW` | Identifies the moment a touch is first registered (transition edge) to prevent continuous state toggling while holding. |
| `lampState = !lampState` | Applies the boolean NOT operator to invert the current logic state (true becomes false, and vice versa). |

## Hardware & Safety Concept: Software Debouncing
Mechanical switches and capacitive pads fluctuate rapidly between HIGH and LOW states for a brief moment when clicked (called "bouncing"). Without a **debounce delay** in code (e.g. `delay(200)`), the microcontroller will interpret this fast fluctuation as multiple separate clicks, causing the lamp to toggle ON and OFF randomly during a single touch.

## Try This! (Challenges)
1. **Beep Confirm**: Sound a buzzer on GP14 for 50 ms every time the lamp changes state.
2. **Double Touch Reset**: Modify the code so that the lamp can only be turned OFF by double-tapping the touch sensor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Lamp toggles rapidly when touched once | Missing state-change detection | Ensure you check `lastTouchState` to capture the rising edge, rather than checking `currentTouchState == HIGH` alone. |

## Mode Notes
This basic state-machine project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [12 - Pico Button Toggle](../../beginner/12-button-toggle.md) (Uno equivalent)
- [21 - Pico Touch Lamp](21-pico-touch-lamp.md)
