# 32 - Pico Potentiometer Dimmer

Dim an external LED brightness by mapping potentiometer analog values to PWM duty cycles.

## Goal
Learn how to read analog voltages and map them to Pulse Width Modulation (PWM) duty cycle values to control light intensity.

## What You Will Build
An LED dimmer control circuit:
- **Potentiometer (GP26)**: Reads 0 to 4095.
- **Red LED (GP15)**: Brightness scales dynamically from fully OFF (0 duty cycle) to fully ON (255 duty cycle) matching the pot dial.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | Pin 1 (VCC) | 3.3V | Power supply |
| Potentiometer | Pin 2 (Wiper) | GP26 | Analog input |
| Potentiometer | Pin 3 (GND) | GND | Ground return |
| Red LED | Anode | GP15 | PWM output pin |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int POT_PIN = 26;
const int LED_PIN = 15;

void setup() {
  pinMode(POT_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int rawValue = analogRead(POT_PIN); // Reads 0 to 4095

  // Map 12-bit input (0-4095) to 8-bit output (0-255)
  int brightness = rawValue / 16; 

  analogWrite(LED_PIN, brightness); // Output PWM signal

  delay(20); // Fast update rate
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Potentiometer**, and **Red LED** onto the canvas.
2. Connect Potentiometer: **Pin 1** to **3V3**, **Pin 2** to **GP26**, **Pin 3** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Move the potentiometer slider and observe the LED brightness fade.

## Expected Output

Terminal:
```
Simulation active. Mapping GP26 input to GP15 PWM output.
```

## Expected Canvas Behavior
| Potentiometer Value | GP26 Read Value | GP15 Output Duty Cycle | LED Visual |
| --- | --- | --- | --- |
| Minimum (0%) | 0 | 0 (0% duty cycle) | OFF |
| Midpoint (50%) | 2048 | 128 (50% duty cycle) | Mid brightness |
| Maximum (100%) | 4095 | 255 (100% duty cycle)| Full brightness |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `rawValue / 16` | Divides 4095 by 16 to scale the range to 255 (fits 8-bit PWM width). |
| `analogWrite(LED_PIN, brightness)` | Starts a PWM square wave on GP15 at the specified duty cycle (0 to 255). |

## Hardware & Safety Concept: PWM Duty Cycle
Microcontrollers cannot output variable analog voltages directly from digital pins. Instead, they use **Pulse Width Modulation (PWM)**. By turning the pin ON and OFF thousands of times per second (frequency), the average voltage delivered to the LED scales with the ratio of ON time to total cycle time (duty cycle). The human eye perceives this fast switching as dimmer light.

## Try This! (Challenges)
1. **Inverted Fader**: Modify the code so that turning the potentiometer to maximum dims the LED, and turning it to minimum makes the LED shine brightest.
2. **Double Fader**: Add a second LED on GP14 and configure them to cross-fade (as LED 1 brightens, LED 2 dims).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED flashes instead of dimming | Delay value is too high | Ensure `delay()` inside `loop()` is kept low (around 10–20 ms) to keep the mapping responsive. |

## Mode Notes
This basic PWM mapping project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [31 - Pico Potentiometer ADC](31-pico-potentiometer-adc.md)
- [49 - Pico LDR Dimmer](49-pico-ldr-dimmer.md)
