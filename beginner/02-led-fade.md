# 02 - LED Fade

Gradually change LED brightness using PWM. Introduction to `analogWrite`.

## Goal
Fade an LED from off to full brightness in step-wise stages using `analogWrite` on a PWM-enabled pin. Learn how microcontrollers simulate analog voltages.

## What You Will Build
An LED connected to pin D9 steps through distinct brightness levels from 0 to 255 and back down, creating a smooth fading effect.

**Why D9?** Arduino pins D3, D5, D6, D9, D10, and D11 support Pulse Width Modulation (PWM). D9 is chosen as a standard PWM channel. Pin D13 does not support PWM on the Uno and cannot be used for fading.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| LED Pin | Arduino Pin | Notes |
| --- | --- | --- |
| A (Anode, longer leg) | D9 | Signal pin - must be PWM-capable (marked ~) |
| C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
// Pin number for the LED (must be a PWM pin)
const int LED_PIN = 9;

void setup() {
  // Configure the pin as an output
  pinMode(LED_PIN, OUTPUT);
  
  // Start serial communication at 9600 baud
  Serial.begin(9600);
  Serial.println("LED Fade ready");
}

void loop() {
  // Step up brightness levels (0 is off, 255 is full brightness)
  analogWrite(LED_PIN, 0);   delay(200);
  analogWrite(LED_PIN, 51);  delay(200);
  analogWrite(LED_PIN, 102); delay(200);
  analogWrite(LED_PIN, 153); delay(200);
  analogWrite(LED_PIN, 204); delay(200);
  analogWrite(LED_PIN, 255); delay(200);
  
  // Step down brightness levels back to off
  analogWrite(LED_PIN, 204); delay(200);
  analogWrite(LED_PIN, 153); delay(200);
  analogWrite(LED_PIN, 102); delay(200);
  analogWrite(LED_PIN, 51);  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **LED** onto the canvas.
3. Connect LED **A** (Anode) to Arduino **D9**.
4. Connect LED **C** (Cathode) to Arduino **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.

## Expected Output

Serial Monitor:
```
LED Fade ready
```

### Expected Canvas Behavior

| Time (ms) | analogWrite Value | LED State |
| --- | --- | --- |
| 0 - 200 | 0 | OFF |
| 200 - 400 | 51 | Very Dim |
| 400 - 600 | 102 | Dim |
| 600 - 800 | 153 | Medium |
| 800 - 1000 | 204 | Bright |
| 1000 - 1200 | 255 | Full Brightness |
| 1200 - 2000 | Steps down (204 -> 153 -> 102 -> 51) | Dimming |

The LED on the canvas fades in and out in a repeating loop.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `const int LED_PIN = 9;` | Declares a constant for pin D9, which supports PWM. |
| `analogWrite(LED_PIN, value)` | Writes an analog value (PWM duty cycle) between 0 (always off) and 255 (always on) to the LED. |
| `delay(200)` | Keeps the LED at the written brightness for 200 milliseconds before moving to the next level. |

## Hardware & Safety Concept: Pulse Width Modulation (PWM)
Microcontrollers are digital devices - they can only output 0V (LOW) or 5V (HIGH). To simulate an analog voltage in between, they use **Pulse Width Modulation (PWM)**. 

PWM works by switching the output pin ON and OFF incredibly fast (approx. 490 Hz on D9). 
- If the pin is ON 10% of the time and OFF 90% of the time, the LED appears dim.
- If the pin is ON 90% of the time and OFF 10% of the time, the LED appears bright.

The percentage of time the signal is HIGH is called the **duty cycle**. An `analogWrite` value of 128 corresponds to a 50% duty cycle.

## Try This! (Challenges)
1. **Smoother Fading**: Increase the number of steps. Add values like 25, 76, 128, 179, and 230 to make the fade feel smoother.
2. **Speed Dial**: Adjust all `delay(200)` values to `delay(50)` to speed up the fade, or `delay(500)` to slow it down.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED blinks fully on and off instead of fading | Pin doesn't support PWM | Ensure the LED is connected to D9, not D13 or other non-PWM pin. |
| LED does not turn on | Reverse polarity | Connect Anode (A) to D9 and Cathode (C) to GND. |
| LED stays dim or doesn't fade | Missing delay steps | Make sure you have `delay(200)` after every single `analogWrite` statement. |

## Mode Notes
These patterns (`pinMode`, `digitalWrite`, `delay`, `Serial.begin`, `Serial.println`, `analogWrite`) are supported by MbedO interpreted mode and are safe for this beginner project. This step-wise implementation avoids using `for` loops, which are not supported by the interpreter.

## Related Projects
- [01 - LED Blink](01-led-blink.md)
- [03 - Breathing LED](03-breathing-led.md)
- [14 - Dial LED Brightness](14-dial-led-brightness.md)
