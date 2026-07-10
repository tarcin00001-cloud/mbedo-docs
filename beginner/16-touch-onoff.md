# 16 - Touch ON/OFF

Control an LED using a TTP223 Capacitive Touch Sensor.

## Goal
Learn how to read an active capacitive touch sensor as a digital input and use it to control an LED output.

## What You Will Build
Touching the active sensor pad on the canvas turns the LED on instantly. Releasing your touch turns the LED off.

**Why D2 and D13?** Pin D2 reads the active-high digital signal from the sensor. Pin D13 drives the visual indicator LED.

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
| Touch Sensor | SIG (Signal) | D2 | Input signal connection |
| Touch Sensor | VCC | 5V | Power supply (5V) |
| Touch Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int TOUCH_PIN = 2;
const int LED_PIN   = 13;

void setup() {
  // Configured as INPUT because TTP223 module outputs a driven 5V (HIGH) signal
  pinMode(TOUCH_PIN, INPUT); 
  pinMode(LED_PIN, OUTPUT);
  
  Serial.begin(9600);
  Serial.println("Touch Sensor Ready");
}

void loop() {
  int touchState = digitalRead(TOUCH_PIN);
  
  if (touchState == HIGH) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("Touch Detected - LED ON");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  delay(100);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **TTP223 Touch Sensor**, and **LED** onto the canvas.
2. Connect Touch Sensor **SIG** to Arduino **D2**, **VCC** to Arduino **5V**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Click the Touch Sensor on the canvas to activate it (simulating touch) and watch the LED respond.

## Expected Output

Terminal:
```
Touch Sensor Ready
Touch Detected - LED ON
Touch Detected - LED ON
...
```

### Expected Canvas Behavior

| Action | Touch Sensor state | SIG Pin Reading (D2) | LED State (D13) |
| --- | --- | --- | --- |
| Unpressed | Idle | LOW (0V) | OFF |
| Pressed / Held | Active | HIGH (5V) | ON |

The LED lights up immediately upon contact with the touch sensor.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `pinMode(TOUCH_PIN, INPUT)` | Configures pin D2 as a standard digital input. We do not use pull-up mode because the sensor chip active-drives the pin to HIGH or LOW. |
| `digitalRead(TOUCH_PIN)` | Reads the active voltage state. Returns `HIGH` when the plate is touched, and `LOW` when idle. |

## Hardware & Safety Concept: Capacitive Sensing
Mechanical switches require physical force to bridge metal contacts. A **capacitive touch sensor** works by measuring changes in electrical capacitance. Your body has a natural capacitance; when your finger gets close to the copper pad on the sensor board, it alters the electrical charge of the pad. The onboard TTP223 chip detects this shift and outputs a digital HIGH signal.

## Try This! (Challenges)
1. **Touch Toggle**: Modify the code using state variables (like [12 - Button Toggle](12-button-toggle.md)) so that touching the sensor once turns the LED on, and it stays on until touched again.
2. **Double Alert tone**: Play a quick sound beep when touch is first detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Incorrect pin mode configuration | Ensure `TOUCH_PIN` is configured as `INPUT` (not `INPUT_PULLUP`). Check that SIG is wired to D2. |
| Sensor doesn't respond | VCC or GND wire missing | The active touch sensor module requires power to operate. Make sure VCC is wired to 5V and GND is connected to ground. |

## Mode Notes
These patterns (`digitalRead` and simple conditional checks) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [11 - Button ON/OFF](11-button-onoff.md)
- [12 - Button Toggle](12-button-toggle.md)
