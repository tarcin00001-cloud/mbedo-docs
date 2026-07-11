# 14 - LED PWM brightness fade (GPIO PWM)

Fade the brightness of an external LED connected to the VEGA ARIES v3 board using Pulse Width Modulation (PWM) state changes without loop blocks.

## Goal
Learn how to use PWM registers (`analogWrite`) to scale power delivery, and implement fading transitions by utilizing the main loop structure as a state incrementer instead of declaring nested loops.

## What You Will Build
An external LED is connected to the VEGA ARIES v3 board on pin `GPIO 15`. The code increments and decrements the duty cycle (brightness) between 0 and 255, causing the LED to smoothly fade in and out.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO 15 | Red | PWM control signal (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 220 Ω resistor in series with the LED to limit current while maintaining the full dynamic range of the PWM signal.

## Code
```cpp
// LED PWM Brightness Fade - VEGA ARIES v3
const int LED_PIN = 15;

// Global state variables for tracking brightness level
int brightness = 0;
int fadeAmount = 5;

void setup() {
  // Configure the PWM-capable pin as an output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Set the current brightness level of the LED
  analogWrite(LED_PIN, brightness);
  
  // Increment brightness for the next loop iteration
  brightness = brightness + fadeAmount;
  
  // Reverse the direction of the fade at the limits (0 and 255)
  if (brightness <= 0 || brightness >= 255) {
    fadeAmount = -fadeAmount;
  }
  
  // Adjust speed of the fade transition (30 milliseconds per step)
  delay(30);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the Resistor to **GPIO 15** and the LED's anode.
3. Wire the LED's cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the external LED widget on the canvas fading in and out smoothly.

## Expected Output
Serial Monitor:
```
System Initialized.
PWM Active on Pin 15.
Duty Cycle: 0 -> 255 -> 0
```

## Expected Canvas Behavior
* The LED slowly brightens to maximum intensity, then slowly dims until it is completely off, repeating the cycle continuously.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogWrite(LED_PIN, brightness)` | Sets the PWM duty cycle on the pin (from 0 to 255). |
| `brightness + fadeAmount` | Modifies the brightness value with each loop execution. |
| `fadeAmount = -fadeAmount` | Negates the step size to reverse the fade direction (brightening vs. dimming). |

## Hardware & Safety Concept: Duty Cycles, PWM Frequency, and Heat Dissipation
* **Duty Cycles & PWM**: Microcontrollers cannot output a variable analog voltage directly from digital pins. Instead, they use Pulse Width Modulation (PWM) to toggle the pin ON and OFF at high speed (usually 490 Hz or 1 kHz). The average voltage is proportional to the duty cycle (e.g. 50% duty cycle yields ~1.65V from a 3.3V pin).
* **Heat Dissipation**: Fading the LED reduces the average current load compared to leaving it continuously ON at full power, protecting both the LED junction and the board regulator from thermal fatigue.

## Try This! (Challenges)
1. **Slower Fade**: Change the delay to 100 milliseconds to slow down the breathing effect.
2. **Exponential Fade**: Modify the increment logic to fade exponentially instead of linearly to match human visual perception.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes instead of fading | Digital output used instead of analogWrite | Ensure you call `analogWrite()` instead of `digitalWrite()`. `digitalWrite()` only supports ON and OFF |
| Fading is choppy | Delay value too large | If the delay is too large (e.g. > 200 ms), the individual steps in brightness will be visible to the eye. Keep the step delay between 15 ms and 50 ms |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [13 - Buzzer warning pulse](13-buzzer-warning-pulse.md)
- [15 - RGB LED breathing cycle](15-rgb-led-breathing-cycle.md) (Next project)
