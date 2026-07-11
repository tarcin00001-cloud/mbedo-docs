# 32 - Potentiometer LED Dimmer (GPIO PWM)

Control the brightness of an external LED using a potentiometer connected to the VEGA ARIES v3 board.

## Goal
Learn how to map an analog sensor input range to a Pulse Width Modulation (PWM) output range to dim an LED smoothly without using nested blocking loops.

## What You Will Build
A potentiometer is connected to analog input pin `ADC0` (`GP26`), and an external LED is connected to `GPIO 15`. The board reads the potentiometer's rotation, maps the 12-bit input (0–4095) to an 8-bit duty cycle (0–255), and outputs a PWM signal to adjust the LED's brightness.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | VCC | 3.3V | Red | Power connection |
| Potentiometer | Output / Signal | ADC0 (GP26) | Yellow | Analog input signal |
| Potentiometer | GND | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | PWM control signal (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Keep the LED's current-limiting resistor connected in series with the anode (+) side to prevent excessive current draw on GPIO 15.

## Code
```cpp
// Potentiometer LED Dimmer - VEGA ARIES v3
const int POT_PIN = GP26;  // ADC0
const int LED_PIN = 15;    // Warning LED pin

void setup() {
  // Configure the LED pin as a digital output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Read the raw 12-bit value from the potentiometer
  int rawValue = analogRead(POT_PIN);
  
  // Scale the 12-bit ADC value (0-4095) to 8-bit PWM (0-255)
  int pwmValue = map(rawValue, 0, 4095, 0, 255);
  
  // Write the PWM duty cycle to the LED
  analogWrite(LED_PIN, pwmValue);
  
  // Minimal delay to smooth output transitions
  delay(50);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, a **Potentiometer**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the Potentiometer's center pin to **GP26 (ADC0)**.
3. Wire the LED's anode (+) through the resistor to **GPIO 15** and its cathode (-) to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Rotate the virtual potentiometer slider and observe the LED changing brightness.

## Expected Output
Serial Monitor:
```
System Initialized.
PWM Active on Pin 15.
```

## Expected Canvas Behavior
* The external LED widget dims or brightens on the canvas in real-time as the potentiometer knob is rotated.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int LED_PIN = 15;` | Defines the pin where the LED is connected. |
| `analogRead(POT_PIN)` | Reads the potentiometer's 12-bit digital value. |
| `map(rawValue, 0, 4095, 0, 255)` | Scales the 12-bit ADC value to an 8-bit duty cycle range. |
| `analogWrite(LED_PIN, pwmValue)` | Writes the scaled duty cycle to the LED pin using PWM. |

## Hardware & Safety Concept: Mapping Control Signals and PWM Current Draw
* **Mapping Control Signals**: Signal scaling is vital in embedded control systems to match the range of a sensor (here, 12-bit input) with the limits of an actuator (here, 8-bit PWM).
* **PWM Current Draw**: Driving an LED with PWM (fast pulsing) reduces overall current consumption compared to continuous DC operation, protecting the THEJAS32 SoC and limiting power dissipation.

## Try This! (Challenges)
1. **Inverse Dimming**: Modify the mapping function so that the LED gets *brighter* as the potentiometer is turned down, and *dimmer* as it is turned up.
2. **Buzzer Tone Pitch Control**: Connect an active buzzer to `GPIO 14` instead of the LED, and map the potentiometer values to different frequency beep rates.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED is either fully ON or fully OFF | Using digital control instead of PWM | Verify the code uses `analogWrite()` and that the LED pin is mapped to a PWM-capable GPIO |
| LED does not light up | Reversed LED orientation | Ensure the LED's longer pin (anode) is connected to the resistor and GPIO 15, and the shorter pin (cathode) is connected to GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - Potentiometer ADC Read (GPIO Analog pin)](31-potentiometer-adc-read.md) (Previous project)
- [33 - LDR light sensor read](33-ldr-light-sensor-read.md) (Next project)
