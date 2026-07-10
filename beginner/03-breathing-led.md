# 03 - Breathing LED

Make an LED smoothly pulse up and down, mimicking a breathing rhythm.

## Goal
Learn how to create smooth transitions in LED brightness using a high-density sequence of PWM outputs.

## What You Will Build
An LED connected to pin D9 smoothly brightens and dims in a cyclic, non-blocking rhythm.

**Why D9?** Fading effects require PWM pins. D9 is used to handle the quick, step-wise power updates cleanly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| LED Pin | Arduino Pin | Notes |
| --- | --- | --- |
| A (Anode, longer leg) | D9 | Signal pin - must be PWM-capable |
| C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int LED_PIN = 9;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Breathing LED ready");
}

void loop() {
  // Fade In
  analogWrite(LED_PIN, 0);   delay(80);
  analogWrite(LED_PIN, 25);  delay(80);
  analogWrite(LED_PIN, 51);  delay(80);
  analogWrite(LED_PIN, 76);  delay(80);
  analogWrite(LED_PIN, 102); delay(80);
  analogWrite(LED_PIN, 128); delay(80);
  analogWrite(LED_PIN, 153); delay(80);
  analogWrite(LED_PIN, 179); delay(80);
  analogWrite(LED_PIN, 204); delay(80);
  analogWrite(LED_PIN, 230); delay(80);
  analogWrite(LED_PIN, 255); delay(80);

  // Fade Out
  analogWrite(LED_PIN, 230); delay(80);
  analogWrite(LED_PIN, 204); delay(80);
  analogWrite(LED_PIN, 179); delay(80);
  analogWrite(LED_PIN, 153); delay(80);
  analogWrite(LED_PIN, 128); delay(80);
  analogWrite(LED_PIN, 102); delay(80);
  analogWrite(LED_PIN, 76);  delay(80);
  analogWrite(LED_PIN, 51);  delay(80);
  analogWrite(LED_PIN, 25);  delay(80);
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
Breathing LED ready
```

### Expected Canvas Behavior

| Time (ms) | Direction | Brightness State |
| --- | --- | --- |
| 0 - 880 | Rising | Dim to full brightness |
| 880 - 1600 | Falling | Full brightness back to dim |
| 1600+ | Cycles | Repeats continuously |

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `analogWrite(LED_PIN, value)` | Sends PWM duty cycle steps of approximately 10% increments (0, 25, 51, etc.) up to 100% (255) and back. |
| `delay(80)` | Holds each brightness level for 80 ms, creating a transition period of about 1.6 seconds per breath. |

## Hardware & Safety Concept: Step Density
In human perception, brightness scaling is non-linear (logarithmic). A sequence of small, equal steps (e.g. increments of 25) with quick delays feels like a smooth, continuous fade. If the step sizes are too large (e.g. jumping from 0 to 128 to 255), the transition will look jerky.

## Try This! (Challenges)
1. **Adjust Breath Rate**: Change the `delay(80)` to `delay(120)` to simulate a slower, relaxed breath, or `delay(40)` for a fast, panting effect.
2. **Asymmetric Breathing**: Change the fade-in delays to `delay(60)` and the fade-out delays to `delay(120)` to create a quick inhalation and a slow exhalation.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes abruptly | LED pin is wrong | Move the LED anode wire to D9 (PWM). |
| LED stays completely on or off | Missing loop cycle delays | Check that a `delay(80)` is present after every single `analogWrite`. |

## Mode Notes
These patterns (`pinMode`, `digitalWrite`, `delay`, `Serial.begin`, `Serial.println`, `analogWrite`) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [02 - LED Fade](02-led-fade.md)
- [04 - RGB Color Mixer](04-rgb-color-mixer.md)
- [14 - Dial LED Brightness](14-dial-led-brightness.md)
