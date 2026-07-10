# 14 - Dial LED Brightness

Control the brightness of an LED using a potentiometer knob.

## Goal
Learn how to read an analog input voltage from a potentiometer (0 to 5V), scale that reading to a PWM duty cycle (0 to 255) using `map()`, and drive LED brightness accordingly.

## What You Will Build
Turning the potentiometer dial on the canvas changes the LED brightness smoothly from fully off to fully bright.

**Why A0 and D9?** Pin A0 connects to the built-in 10-bit Analog-to-Digital Converter (ADC). Pin D9 is a dedicated PWM output channel required to drive varying LED brightness.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | 1 | 5V | Positive voltage reference |
| Potentiometer | 2 (Wiper) | A0 | Analog signal connection |
| Potentiometer | 3 | GND | Ground reference |
| LED | A (Anode, longer leg) | D9 | Output signal connection (must be PWM pin) |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int POT_PIN = A0;
const int LED_PIN = 9; // Must be a PWM pin (3, 5, 6, 9, 10, 11)

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Dial Brightness Ready");
}

void loop() {
  // Read potentiometer value (returns 0 to 1023)
  int val = analogRead(POT_PIN);
  
  // Scale value to LED duty cycle range (0 to 255)
  int brightness = map(val, 0, 1023, 0, 255);
  
  // Write brightness duty cycle to LED
  analogWrite(LED_PIN, brightness);
  
  Serial.print("Pot: ");
  Serial.print(val);
  Serial.print(" - Brightness: ");
  Serial.println(brightness);
  
  delay(100); // 100 ms refresh delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Potentiometer**, and **LED** onto the canvas.
2. Connect Potentiometer **pin 1** to Arduino **5V**, **pin 2 (wiper)** to Arduino **A0**, and **pin 3** to Arduino **GND**.
3. Connect LED **A** to Arduino **D9** (a PWM pin) and LED **C** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the Potentiometer to open its properties, adjust the slider/knob value, and watch the LED brightness change.

## Expected Output

Serial Monitor:
```
Dial Brightness Ready
Pot: 0 - Brightness: 0
Pot: 512 - Brightness: 127
Pot: 1023 - Brightness: 255
...
```

### Expected Canvas Behavior

| Potentiometer Angle | Wiper Voltage | Pin A0 Reading | mapped Duty Cycle | LED State |
| --- | --- | --- | --- | --- |
| 0% (Counter-clockwise) | 0.0V | 0 | 0 | OFF |
| 50% (Middle) | 2.5V | 512 | 127 | Medium Bright |
| 100% (Clockwise) | 5.0V | 1023 | 255 | Full Bright |

The LED on the canvas alters brightness smoothly as the potentiometer dial is turned.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `analogRead(POT_PIN)` | Reads the voltage on pin A0. The Arduino ADC converts this voltage (0 to 5V) into a whole number between `0` (for 0V) and `1023` (for 5V). |
| `map(val, 0, 1023, 0, 255)` | Scales the 10-bit input scale (0-1023) down to the 8-bit output scale (0-255) needed for PWM output. |
| `analogWrite(LED_PIN, brightness)` | Pushes a PWM wave of the mapped duty cycle to D9, altering the power delivered to the LED. |

## Hardware & Safety Concept: Analog vs Digital Signals
* A **digital signal** has only two states: ON (5V) or OFF (0V).
* An **analog signal** can vary continuously to any voltage between 0V and 5V.
A potentiometer acts as an adjustable voltage divider. Turning the knob sweeps the center wiper pin across a resistive track, outputting a variable voltage. The Arduino ADC measures this voltage and maps it to a digital representation.

## Try This! (Challenges)
1. **Reverse Knob Direction**: Modify the `map()` arguments so that turning the potentiometer clockwise *dims* the LED instead of brightening it. (Hint: change the target range to `255, 0`).
2. **Visual Step Meter**: Print a series of dashes to the Terminal representing the dial level.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays fully on or off | LED is on a non-PWM pin | Move the LED anode wire from its current pin to D9, D10, or D11. |
| Potentiometer reads constant 0 or 1023 | Outer pins not connected to 5V/GND | Make sure potentiometer pin 1 connects to 5V and pin 3 connects to GND. |

## Mode Notes
These patterns (`analogRead` mapped directly to `analogWrite`) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [02 - LED Fade](02-led-fade.md)
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
