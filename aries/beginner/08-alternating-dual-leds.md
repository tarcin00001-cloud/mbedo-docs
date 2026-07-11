# 08 - Alternating Dual LEDs (flashing)

Flash two external LEDs in an alternating pattern on the VEGA ARIES v3 board to simulate a railway crossing signal.

## Goal
Learn how to coordinate multiple digital output channels in opposite states to create alternating visual patterns.

## What You Will Build
Two external LEDs are wired to the VEGA ARIES v3 board (LED 1 on `GPIO 15` and LED 2 on `GPIO 13`). The program toggles them so that when one LED is ON, the other is OFF, switching states every 500 milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 2x LEDs (Red & Yellow) | `led` | Yes | Yes |
| 2x 220 Ω Resistors | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED 1 (Red) | Anode (+) | GPIO 15 | Red | Control signal for LED 1 (via resistor) |
| LED 2 (Yellow) | Anode (+) | GPIO 13 | Yellow | Control signal for LED 2 (via resistor) |
| Both LEDs | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with each LED's positive terminal to protect the board's GPIO pins.

## Code
```cpp
// Alternating Dual LEDs - VEGA ARIES v3
const int LED1_PIN = 15;
const int LED2_PIN = 13;

void setup() {
  // Configure both pins as digital outputs
  pinMode(LED1_PIN, OUTPUT);
  pinMode(LED2_PIN, OUTPUT);
}

void loop() {
  // State A: LED 1 is ON, LED 2 is OFF
  digitalWrite(LED1_PIN, HIGH);
  digitalWrite(LED2_PIN, LOW);
  delay(500);
  
  // State B: LED 1 is OFF, LED 2 is ON
  digitalWrite(LED1_PIN, LOW);
  digitalWrite(LED2_PIN, HIGH);
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **2 LEDs**, and **2 Resistors** onto the canvas.
2. Wire the first LED's anode to **GPIO 15** (through resistor) and the second LED's anode to **GPIO 13** (through resistor).
3. Connect both cathodes to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the two LED widgets flashing alternately on the canvas.

## Expected Output
Serial Monitor:
```
System Initialized.
Alternating States:
  LED 1: ON  | LED 2: OFF
  LED 1: OFF | LED 2: ON
```

## Expected Canvas Behavior
* LED 1 lights up for 0.5 seconds while LED 2 goes dark, then LED 1 goes dark while LED 2 lights up for 0.5 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED1_PIN, OUTPUT)` | Sets up the first digital pin. |
| `digitalWrite(LED1_PIN, HIGH)` | Drives pin 15 HIGH while driving pin 13 LOW, establishing the first state. |
| `digitalWrite(LED1_PIN, LOW)` | Swaps the pin states to establish the alternating pattern. |

## Hardware & Safety Concept: Current Draw and Ground Noise
* **Current Draw**: When switching multiple output pins at the same time, the total current drawn from the microcontroller's power supply rail increases. Ensure the total load does not exceed the absolute maximum limit of the SoC (typically 100–150 mA across all pins).
* **Ground Noise**: Alternating switching loads can induce noise on the ground rail, which can affect analog-to-digital converter (ADC) readings. Always route power wires directly to prevent ground bounce.

## Try This! (Challenges)
1. **Asymmetric Blink**: Adjust the delays so one LED flashes quickly and the other flashes slowly.
2. **Police Flash**: Generate double flashes on each side (LED 1 ON/OFF twice, then LED 2 ON/OFF twice) to simulate emergency vehicle lights.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Only one LED flashes | Code logic error | Ensure both `LED1_PIN` and `LED2_PIN` are configured in `setup()`, and that both pins are updated in both parts of `loop()` |
| Both LEDs flash together | Pins tied to same line | Verify that your wires connect to separate GPIO pins (15 and 13) and are not shorted on the breadboard |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [07 - External LED Blink (pin-tied)](07-external-led-blink.md)
- [09 - LED SOS Signal generator](09-led-sos-signal-generator.md) (Next project)
