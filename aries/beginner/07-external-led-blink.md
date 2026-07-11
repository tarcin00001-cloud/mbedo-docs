# 07 - External LED Blink (pin-tied)

Blink an external LED connected to a digital output pin on the VEGA ARIES v3 board.

## Goal
Learn how to interface external discrete components (like LEDs), understand current-limiting resistor wiring, and control digital output pins.

## What You Will Build
An external LED is wired to the VEGA ARIES v3 board on pin `GPIO 15`. The LED is toggled ON and OFF every 1000 milliseconds, creating a flashing status light.

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
| LED | Anode (+) | GPIO 15 | Red | Digital control signal (through resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED's anode (+) to prevent the LED from burning out or drawing too much current from the GPIO pin.

## Code
```cpp
// External LED Blink - VEGA ARIES v3
const int LED_PIN = 15;

void setup() {
  // Configure the warning LED pin as a digital output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Turn the external LED ON
  digitalWrite(LED_PIN, HIGH);
  delay(1000);
  
  // Turn the external LED OFF
  digitalWrite(LED_PIN, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire one end of the Resistor to **GPIO 15** and the other end to the LED's anode.
3. Wire the LED's cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Observe the external LED widget blinking on the canvas.

## Expected Output
Serial Monitor:
```
System Initialized.
GPIO 15 State: HIGH
GPIO 15 State: LOW
```

## Expected Canvas Behavior
* The external LED widget lights up for 1 second, dims for 1 second, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int LED_PIN = 15` | Sets a symbolic constant name for the external LED pin (GPIO 15). |
| `pinMode(LED_PIN, OUTPUT)` | Defines pin 15 as a digital output. |
| `digitalWrite(LED_PIN, HIGH)` | Drives the pin HIGH, allowing current to flow and lighting up the LED. |

## Hardware & Safety Concept: Current-Limiting Resistors and Pin Sourcing
* **Current-Limiting Resistors**: LEDs are diodes with very low internal resistance when forward-biased. Connecting an LED directly between a 3.3V GPIO pin and GND without a series resistor will create a near short-circuit, which can destroy the LED or damage the microcontroller pin.
* **Pin Sourcing**: The THEJAS32 SoC on the VEGA ARIES v3 board has a maximum current capability per GPIO pin (typically 8–12 mA). Always use a resistor (minimum 220 Ω for 3.3V) to limit the current draw to a safe level (around 5–10 mA).

## Try This! (Challenges)
1. **SOS Strobe**: Change the delays to 100 milliseconds to simulate a strobe warning pattern.
2. **Breathing Delay**: Make the LED stay ON for 2 seconds and OFF for 500 milliseconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED does not light up | Wrong orientation | LEDs are polarized. The longer leg (anode) must connect to the positive voltage (GPIO 15) and the shorter leg (cathode) to GND |
| LED is extremely dim | Resistor value too high | Ensure you are using a 220 Ω resistor, not a 10 kΩ or 100 kΩ resistor |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [06 - Active Buzzer Alarm sequence](06-active-buzzer-alarm-sequence.md)
- [08 - Alternating Dual LEDs (flashing)](08-alternating-dual-leds.md) (Next project)
