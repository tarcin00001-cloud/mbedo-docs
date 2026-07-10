# 27 - Touch Lamp

Build a touch-sensitive lamp that toggles ON and OFF with each touch.

## Goal
Learn how to apply toggle state logic to a capacitive touch sensor input, turning it into a latching ON/OFF switch.

## What You Will Build
Touching the touch sensor once turns the LED on. The LED stays on when you remove your hand. Touching the sensor a second time turns the LED off.

**Why D2 and D13?** Pin D2 reads the transition edge of the touch sensor. Pin D13 drives the visual indicator LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| TTP223 Touch | `ttp223_touch` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Touch Sensor | SIG | D2 | Input signal connection |
| Touch Sensor | VCC | 5V | Power supply (5V) |
| Touch Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int TOUCH_PIN = 2;
const int LED_PIN   = 13;

int lampState = LOW;       // Track if the lamp is ON or OFF
int lastTouchState = LOW;  // Track the previous reading of the touch sensor

void setup() {
  pinMode(TOUCH_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, lampState);
  
  Serial.begin(9600);
  Serial.println("Touch Lamp Ready");
}

void loop() {
  int touchState = digitalRead(TOUCH_PIN);

  // Detect state transition from LOW (no touch) to HIGH (touched)
  if (touchState == HIGH && lastTouchState == LOW) {
    if (lampState == LOW) {
      lampState = HIGH;
      Serial.println("Lamp - ON");
    } else {
      lampState = LOW;
      Serial.println("Lamp - OFF");
    }
    digitalWrite(LED_PIN, lampState);
    delay(200); // Debounce delay
  }
  
  lastTouchState = touchState;
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **TTP223 Touch Sensor**, and **LED** onto the canvas.
2. Connect Touch Sensor **SIG** to Arduino **D2**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the Touch Sensor once to turn the LED on, and click it again to turn it off.

## Expected Output

Terminal:
```
Touch Lamp Ready
Lamp - ON
Lamp - OFF
...
```

### Expected Canvas Behavior

| Touch Action | lastTouchState | touchState | lampState Result | LED State (D13) |
| --- | --- | --- | --- | --- |
| Default | LOW | LOW | LOW | LOW - OFF |
| 1st Touch | LOW | HIGH | HIGH | HIGH - ON |
| Hand Removed | HIGH | LOW | HIGH (no change) | HIGH - ON |
| 2nd Touch | LOW | HIGH | LOW | LOW - OFF |

The lamp toggles state exactly on the rising transition edge of the touch event.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (touchState == HIGH && lastTouchState == LOW)` | Checks for a rising edge (the exact millisecond contact begins). This prevents the lamp from toggling repeatedly if you hold your finger on the sensor. |

## Hardware & Safety Concept: Capacitive Toggle Switch
In consumer electronics, touch lamps use a conductive metal plate connected to a specialized sensing chip. The TTP223 chip handles all the high-frequency measurements to detect changes in electrical capacitance and converts this into a simple, clean digital HIGH signal when touched, making code implementation direct.

## Try This! (Challenges)
1. **Brightness presets**: Use D9 for the LED and modify the code to cycle through Dim (50), Medium (150), Full (255), and OFF on consecutive touches.
2. **Double chirp**: Add a Buzzer to D8 and code a double chirp sound whenever the lamp turns ON.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Lamp toggles rapidly when touched | Missing state transition check | Ensure you have `lastTouchState = touchState` at the end of the loop and you are checking `lastTouchState == LOW`. |
| LED doesn't light up | Wiring issue | Make sure Touch Sensor SIG is connected to D2, VCC to 5V, and GND to ground. |

## Mode Notes
These patterns (state transitions and digital toggles) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [12 - Button Toggle](12-button-toggle.md)
- [16 - Touch ON/OFF](16-touch-onoff.md)
