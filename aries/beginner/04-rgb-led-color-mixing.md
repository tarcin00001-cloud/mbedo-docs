# 04 - RGB LED Color Mixing (alternating delays)

Mix colors on the onboard RGB LED of the VEGA ARIES v3 board by toggling the Red, Green, and Blue channels.

## Goal
Learn how to control multiple digital output channels simultaneously to mix red, green, and blue light components to produce secondary colors (Yellow, Cyan, Magenta, White).

## What You Will Build
The onboard RGB LED's Red, Green, and Blue pins (`LED_R`, `LED_G`, `LED_B`) are toggled in combination to cycle through Red, Green, Blue, Yellow (Red+Green), Cyan (Green+Blue), Magenta (Red+Blue), and White (Red+Green+Blue) with alternating delays.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Onboard RGB LED | Built-in | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED (Red) | LED_R | LED_R (GPIO 23) | Internal | Pre-wired channel |
| Onboard LED (Green) | LED_G | LED_G (GPIO 24) | Internal | Pre-wired channel |
| Onboard LED (Blue) | LED_B | LED_B (GPIO 25) | Internal | Pre-wired channel |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The onboard RGB LED requires no external wiring since all three channels are internally linked to the microcontroller pins.

## Code
```cpp
// RGB LED Color Mixing - VEGA ARIES v3
void setup() {
  // Configure all three channels as digital outputs
  pinMode(LED_R, OUTPUT);
  pinMode(LED_G, OUTPUT);
  pinMode(LED_B, OUTPUT);
}

void loop() {
  // 1. RED
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, LOW);
  digitalWrite(LED_B, LOW);
  delay(1000);
  
  // 2. GREEN
  digitalWrite(LED_R, LOW);
  digitalWrite(LED_G, HIGH);
  digitalWrite(LED_B, LOW);
  delay(1000);
  
  // 3. BLUE
  digitalWrite(LED_R, LOW);
  digitalWrite(LED_G, LOW);
  digitalWrite(LED_B, HIGH);
  delay(1000);
  
  // 4. YELLOW (Red + Green)
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, HIGH);
  digitalWrite(LED_B, LOW);
  delay(1000);
  
  // 5. CYAN (Green + Blue)
  digitalWrite(LED_R, LOW);
  digitalWrite(LED_G, HIGH);
  digitalWrite(LED_B, HIGH);
  delay(1000);
  
  // 6. MAGENTA (Red + Blue)
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, LOW);
  digitalWrite(LED_B, HIGH);
  delay(1000);
  
  // 7. WHITE (Red + Green + Blue)
  digitalWrite(LED_R, HIGH);
  digitalWrite(LED_G, HIGH);
  digitalWrite(LED_B, HIGH);
  delay(1000);
  
  // 8. OFF (Black)
  digitalWrite(LED_R, LOW);
  digitalWrite(LED_G, LOW);
  digitalWrite(LED_B, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Select the board and verify that the onboard RGB LED is visible.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the RGB LED widget on the board cycle through Red, Green, Blue, Yellow, Cyan, Magenta, White, and OFF.

## Expected Output
Serial Monitor:
```
System Initialized.
Color State: Red
Color State: Green
Color State: Blue
Color State: Yellow
Color State: Cyan
Color State: Magenta
Color State: White
Color State: Off
```

## Expected Canvas Behavior
* The onboard RGB LED changes colors every second following the sequence.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(...)` | Configures the physical GPIO pins 23, 24, and 25 to act as digital outputs. |
| `digitalWrite(LED_R, HIGH)` | Activates the Red channel. |
| `digitalWrite(LED_G, HIGH)` | Mixing Red and Green channels produces Yellow light. |

## Hardware & Safety Concept: Additive Color Mixing and Thermal Constraints
* **Additive Color Mixing**: RGB LEDs work on the principle of additive color mixing. By combining three primary wavelengths (Red, Green, Blue) at full intensity, they appear to the human eye as secondary colors or White.
* **Thermal Constraints**: Running all three LED channels at full power (producing White) draws the maximum current and generates the most heat. The onboard current-limiting resistors are selected to keep power dissipation within safe limits even during continuous White emission.

## Try This! (Challenges)
1. **Fast Strobe**: Reduce all delays to 200 milliseconds to create a rapid color strobing pattern.
2. **Custom Color sequence**: Re-order the states to transition smoothly from cool colors (Blue, Cyan, Green) to warm colors (Yellow, Red, Magenta).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED displays Red instead of Yellow | Green pin connection error | Verify that `LED_G` is set `HIGH` in your code during the yellow phase |
| Compile error | Variable name typos | Verify that all pin labels (`LED_R`, `LED_G`, `LED_B`) are spelled correctly and in uppercase |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [03 - Onboard RGB LED Blue toggle](03-onboard-rgb-led-blue-toggle.md)
- [05 - Active Buzzer Beep](05-active-buzzer-beep.md) (Next project)
