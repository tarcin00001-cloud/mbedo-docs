# 09 - LED SOS Signal generator

Generate a visual Morse code SOS distress signal (`. . .   - - -   . . .`) using an external LED on the VEGA ARIES v3 board.

## Goal
Learn how to implement complex timed signaling patterns using sequential output statements and delay intervals.

## What You Will Build
An external LED connected to pin `GPIO 15` is programmed to flash out the SOS Morse code signal sequence: three short pulses (Dots), three long pulses (Dashes), and three short pulses (Dots), followed by a 2-second pause.

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
| LED | Anode (+) | GPIO 15 | Red | Control signal for LED (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED's positive terminal to protect the board's GPIO pins.

## Code
```cpp
// LED SOS Signal Generator - VEGA ARIES v3
const int LED_PIN = 15;

void setup() {
  // Configure the LED pin as a digital output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // ==========================================
  // Letter S: Three Dots (. . .)
  // ==========================================
  // Dot 1
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dot 2
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dot 3
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  
  // Gap between S and O (600 ms total, 200 ms already elapsed)
  delay(400);
  
  // ==========================================
  // Letter O: Three Dashes (- - -)
  // ==========================================
  // Dash 1
  digitalWrite(LED_PIN, HIGH);
  delay(600);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dash 2
  digitalWrite(LED_PIN, HIGH);
  delay(600);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dash 3
  digitalWrite(LED_PIN, HIGH);
  delay(600);
  digitalWrite(LED_PIN, LOW);
  
  // Gap between O and S (600 ms total, 200 ms already elapsed)
  delay(400);
  
  // ==========================================
  // Letter S: Three Dots (. . .)
  // ==========================================
  // Dot 1
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dot 2
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  delay(200);
  
  // Dot 3
  digitalWrite(LED_PIN, HIGH);
  delay(200);
  digitalWrite(LED_PIN, LOW);
  
  // Gap between SOS repetitions
  delay(2000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the Resistor to **GPIO 15** and the LED's anode.
3. Wire the LED's cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the LED widget flashing out the SOS pattern.

## Expected Output
Serial Monitor:
```
System Initialized.
Morse Code: S (. . .)
Morse Code: O (- - -)
Morse Code: S (. . .)
Distress Cycle Complete. Repeating...
```

## Expected Canvas Behavior
* The LED flashes in three rapid bursts (Dots), three longer pulses (Dashes), and three rapid bursts (Dots), pauses for 2 seconds, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `delay(200)` | Defines the unit time (Dot duration and spacing between elements). |
| `delay(600)` | Defines the Dash duration (3 times the Dot duration). |
| `delay(2000)` | Establishes a 2-second pause before repeating the distress signal. |

## Hardware & Safety Concept: Morse Code Signatures and LED Overheating
* **Morse Code Signatures**: Visual distress signals follow standardized timing rules where a Dash is three times longer than a Dot, and the space between letters is equal to one Dash.
* **LED Overheating**: Since this program alternates the LED states with balanced delays, the duty cycle is relatively low (~30%), meaning the LED is not powered continuously. This prevents heat buildup in the LED junction.

## Try This! (Challenges)
1. **Buzzer Integration**: Connect a buzzer (GPIO 14) and program it to play the sound matching the LED flashes to create an audible SOS distress signal.
2. **Speed adjustment**: Speed up the signal transmission by reducing the Dot duration to 100 milliseconds and scaling all other delays proportionally.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The LED is stuck ON | Code hangs or logic error | Verify that all `digitalWrite(..., LOW)` commands are present in the sequence. A missing state update will leave the LED powered |
| Flashing pattern is uneven | Math error in delays | Ensure you follow the timing ratios: Dots = 200 ms, Dashes = 600 ms, spacing = 200 ms |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - Alternating Dual LEDs (flashing)](08-alternating-dual-leds.md)
- [10 - Relay Module Switch](10-relay-module-switch.md) (Next project)
