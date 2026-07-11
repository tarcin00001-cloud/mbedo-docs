# 116 - LED Bar Graph Level Indicator

Build an analog level meter using a potentiometer to drive an 8-pin LED bar graph display on the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs, map voltage ranges to linear step levels, and update multiple digital outputs line-by-line without loops.

## What You Will Build
A visual level indicator (like a volume unit meter). Rotating the potentiometer dial scales the analog voltage input, lighting up more segments on the 8-LED bar graph as the dial turns.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| 8-Segment LED Bar Graph | `bar_graph` | Yes | Yes |
| 220-ohm Resistors | `resistor` | Yes | Yes (8 required) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3V3 | Red | Analog voltage supply |
| Potentiometer | Pin 2 (Wiper) | ADC0 (GP26) | White | Analog output signal |
| Potentiometer | Pin 3 (GND) | GND | Black | Ground reference |
| LED Bar Graph | Pin 1 (LED 1 Anode) | GPIO 2 | Orange | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 2 (LED 2 Anode) | GPIO 3 | Yellow | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 3 (LED 3 Anode) | GPIO 4 | Green | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 4 (LED 4 Anode) | GPIO 5 | Blue | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 5 (LED 5 Anode) | GPIO 6 | Purple | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 6 (LED 6 Anode) | GPIO 7 | White | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 7 (LED 7 Anode) | GPIO 8 | Gray | Driven through 220-ohm resistor |
| LED Bar Graph | Pin 8 (LED 8 Anode) | GPIO 9 | Brown | Driven through 220-ohm resistor |
| LED Bar Graph | Cathodes (1-8) | GND | Black | Shared Ground reference |

> **Wiring tip:** When using an LED bar graph, each individual segment acts as a standard LED. Connect a 220-ohm current-limiting resistor in series with each segment anode to prevent pin damage.

## Code
```cpp
const int POT_PIN = 26; // ADC0 is GP26

// LED Pin definitions
const int LED1 = 2;
const int LED2 = 3;
const int LED3 = 4;
const int LED4 = 5;
const int LED5 = 6;
const int LED6 = 7;
const int LED7 = 8;
const int LED8 = 9;

void setup() {
  pinMode(LED1, OUTPUT);
  pinMode(LED2, OUTPUT);
  pinMode(LED3, OUTPUT);
  pinMode(LED4, OUTPUT);
  pinMode(LED5, OUTPUT);
  pinMode(LED6, OUTPUT);
  pinMode(LED7, OUTPUT);
  pinMode(LED8, OUTPUT);
}

void loop() {
  int val = analogRead(POT_PIN);

  // Write outputs line-by-line to strictly avoid the "NO loops" constraint
  // Maps 12-bit ADC range (0-4095) to 8 discrete level thresholds
  digitalWrite(LED1, (val >= 500) ? HIGH : LOW);
  digitalWrite(LED2, (val >= 1000) ? HIGH : LOW);
  digitalWrite(LED3, (val >= 1500) ? HIGH : LOW);
  digitalWrite(LED4, (val >= 2000) ? HIGH : LOW);
  digitalWrite(LED5, (val >= 2500) ? HIGH : LOW);
  digitalWrite(LED6, (val >= 3000) ? HIGH : LOW);
  digitalWrite(LED7, (val >= 3500) ? HIGH : LOW);
  digitalWrite(LED8, (val >= 4000) ? HIGH : LOW);

  delay(50); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Potentiometer**, and **8-Segment LED Bar Graph** components onto the canvas.
2. Wire the Potentiometer: **Pin 1** to **3V3**, **Wiper** to **ADC0 (GP26)**, and **Pin 3** to **GND**.
3. Wire the Bar Graph anodes to **GPIO 2-9** through 220-ohm resistors. Wire cathodes to **GND**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Potentiometer: 2048, Active LEDs: 4
```

## Expected Canvas Behavior
* Adjusting the simulated potentiometer dial sweeps the lit segments on the LED bar graph.
* Turning the dial fully counter-clockwise turns all segments off.
* Turning the dial fully clockwise lights up all 8 segments.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(LED1, OUTPUT)` | Configures pin GPIO 2 as digital output segment. |
| `analogRead(POT_PIN)` | Samples the voltage divider from the potentiometer wiper. |
| `(val >= 500) ? HIGH : LOW` | Ternary operator verifying if reading is high enough to light the segment. |
| `digitalWrite(LED8, ...)` | Sets the final segment output high when voltage reaches maximum limit. |

## Hardware & Safety Concept
* **Pin Sourcing Limits**: The RISC-V microcontroller on the ARIES board has a maximum current output limit per GPIO pin (typically 8mA–12mA). Operating all 8 segments of the bar graph simultaneously requires current-limiting resistors (like 220-ohms or 330-ohms) to ensure the total cumulative current does not exceed safe operating limits.
* **Resistor Color Code Identification**: A 220-ohm resistor has color bands: Red, Red, Brown, Gold. Using incorrect resistor values (e.g. 10k-ohm instead of 220-ohm) will make the LEDs very dim or not light up at all.

## Try This! (Challenges)
1. **Dot Indicator Mode**: Modify the logic so that only a single LED segments lights up corresponding to the level (i.e. if value is around 2000, only LED4 lights up, and LEDs 1-3, 5-8 stay off).
2. **Reverse Direction Bar**: Re-map the pin output states so that rotating the potentiometer clockwise turns LEDs off instead of on (inverse level meter).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| One segment does not light up at all | Blown LED or bad wiring | Check that the wire and resistor connections for that specific pin are secure and check the LED polarity. |
| Bar graph segment order is scrambled | Pins wired in wrong sequence | Re-verify that LED1 to LED8 connect sequentially to GPIO 2 to 9. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [14 - LED PWM Brightness Fade](../beginner/14-led-pwm-brightness-fade.md)
- [67 - Potentiometer Speed DC Motor (with LCD)](67-potentiometer-speed-dc-motor.md)
