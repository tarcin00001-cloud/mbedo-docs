# 15 - RGB LED breathing cycle

Fade the onboard RGB LED of the VEGA ARIES v3 board in a sequential "breathing" cycle (Red, Green, Blue) using a loop-free state machine.

## Goal
Learn how to implement complex state machines to control multi-channel outputs sequentially without using nested loop structures.

## What You Will Build
The onboard RGB LED's Red, Green, and Blue channels are faded in and out sequentially: Red fades up and down, then Green fades up and down, then Blue fades up and down, creating a breathing color spectrum.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Onboard RGB LED | Built-in | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED (Red) | LED_R | LED_R (GPIO 23) | Internal | Pre-routed channel |
| Onboard LED (Green) | LED_G | LED_G (GPIO 24) | Internal | Pre-routed channel |
| Onboard LED (Blue) | LED_B | LED_B (GPIO 25) | Internal | Pre-routed channel |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The onboard RGB LED requires no external wiring since all three channels are internally linked to the microcontroller pins.

## Code
```cpp
// Onboard RGB LED Breathing Cycle - VEGA ARIES v3
int state = 0; // 0 = Red up, 1 = Red down, 2 = Green up, 3 = Green down, 4 = Blue up, 5 = Blue down
int val = 0;   // Current PWM intensity value

void setup() {
  // Configure all three channels as digital outputs
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);
  pinMode(LED_B, OUTPUT);
  
  // Set initial states to completely dark
  analogWrite(LED_R, 0);
  analogWrite(LED_G, 0);
  analogWrite(LED_B, 0);
}

void loop() {
  if (state == 0) {
    // Red Channel Fading Up
    analogWrite(LED_R, val);
    analogWrite(LED_G, 0);
    analogWrite(LED_B, 0);
    val = val + 5;
    if (val >= 255) {
      val = 255;
      state = 1; // Transition to Red fading down
    }
  } 
  else if (state == 1) {
    // Red Channel Fading Down
    analogWrite(LED_R, val);
    analogWrite(LED_G, 0);
    analogWrite(LED_B, 0);
    val = val - 5;
    if (val <= 0) {
      val = 0;
      state = 2; // Transition to Green fading up
    }
  } 
  else if (state == 2) {
    // Green Channel Fading Up
    analogWrite(LED_R, 0);
    analogWrite(LED_G, val);
    analogWrite(LED_B, 0);
    val = val + 5;
    if (val >= 255) {
      val = 255;
      state = 3; // Transition to Green fading down
    }
  } 
  else if (state == 3) {
    // Green Channel Fading Down
    analogWrite(LED_R, 0);
    analogWrite(LED_G, val);
    analogWrite(LED_B, 0);
    val = val - 5;
    if (val <= 0) {
      val = 0;
      state = 4; // Transition to Blue fading up
    }
  } 
  else if (state == 4) {
    // Blue Channel Fading Up
    analogWrite(LED_R, 0);
    analogWrite(LED_G, 0);
    analogWrite(LED_B, val);
    val = val + 5;
    if (val >= 255) {
      val = 255;
      state = 5; // Transition to Blue fading down
    }
  } 
  else if (state == 5) {
    // Blue Channel Fading Down
    analogWrite(LED_R, 0);
    analogWrite(LED_G, 0);
    analogWrite(LED_B, val);
    val = val - 5;
    if (val <= 0) {
      val = 0;
      state = 0; // Loop back to Red fading up
    }
  }
  
  // Controls the speed of the breathing effect (20 ms per step)
  delay(20);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Select the board and verify that the onboard RGB LED is visible.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the RGB LED widget on the board fade smoothly through Red, Green, and Blue breathing phases.

## Expected Output
Serial Monitor:
```
System Initialized.
State: Red Breathing Cycle
State: Green Breathing Cycle
State: Blue Breathing Cycle
```

## Expected Canvas Behavior
* The onboard RGB LED breathes in Red, fades to dark, breathes in Green, fades to dark, breathes in Blue, fades to dark, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `int state = 0` | Holds the active phase index of the color cycle. |
| `val = val + 5` | Smoothly increments the PWM value. |
| `state = 2` | Swaps the color channel target once the previous channel is fully dimmed. |

## Hardware & Safety Concept: Additive Mixing and Current Balance
* **Additive Mixing**: By using state variables instead of nested loops, we ensure that only one channel runs its fade cycle at a time, keeping overall current consumption low and stable.
* **Current Balance**: Since only one LED channel is active at any given millisecond, the voltage rail remains exceptionally stable, eliminating transient voltage drops.

## Try This! (Challenges)
1. **Overlap Fading**: Modify the state machine transitions so one color begins to fade up while the previous color is fading down (cross-fade).
2. **Speed Controls**: Set different delay rates for the different color channels to make Red breathe faster than Blue.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED snaps ON instead of fading | Step size too large | Ensure `val` changes by small steps (like 5), and that the delay is around 15–30 ms |
| The color order is wrong | State transition error | Verify the state variable assignment chain (0 -> 1 -> 2 -> 3 -> 4 -> 5 -> 0) matches the code comments |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [14 - LED PWM brightness fade (GPIO PWM)](14-led-pwm-brightness-fade.md)
- [16 - Push Button LED toggle (internal pull-up)](16-push-button-led-toggle.md) (Next project)
