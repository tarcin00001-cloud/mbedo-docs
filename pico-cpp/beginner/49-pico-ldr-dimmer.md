# 49 - Pico LDR Dimmer

Automatically adjust an LED's brightness based on ambient light levels (solar-charging indicator style).

## Goal
Learn how to map variable light intensity readings from an LDR to proportional PWM outputs to control LED brightness dynamically.

## What You Will Build
An ambient light responder:
- **LDR Sensor (GP26)**: Measures room light levels (0 to 4095).
- **Warning LED (GP15)**: Automatically brightens in dark environments and dims in bright environments.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| LDR (Photoresistor) | `photoresistor` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (voltage divider) |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Power supply |
| LDR | Pin 2 | GP26 | Analog input (wire 10k resistor from GP26 to GND) |
| LDR | Pin 2 | GND | Ground return via resistor |
| Red LED | Anode | GP15 | PWM output |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int LDR_PIN = 26;
const int LED_PIN = 15;

void setup() {
  pinMode(LDR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  int lightLevel = analogRead(LDR_PIN); // Reads 0 to 4095

  // Map 12-bit input (0-4095) to 8-bit output (0-255)
  // Invert the value so it gets brighter in darkness:
  int brightness = (4095 - lightLevel) / 16;

  // Constrain brightness value to valid PWM limits
  if (brightness < 0) {
    brightness = 0;
  }
  if (brightness > 255) {
    brightness = 255;
  }

  analogWrite(LED_PIN, brightness);

  delay(50);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **LDR Sensor**, and **Red LED** onto the canvas.
2. Connect LDR: **Pin 1** to **3V3**, **Pin 2** to **GP26** (with 10k resistor to GND).
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Drag the LDR light slider on canvas to dark, and watch the LED glow brighter.

## Expected Output

Terminal:
```
Simulation active. GP26 LDR fader loop running.
```

## Expected Canvas Behavior
| Ambient Light | GP26 Read Value | GP15 PWM Duty Cycle | LED Visual |
| --- | --- | --- | --- |
| Bright Room | > 3500 | < 30 | Very Dim / OFF |
| Dim Room | 2000 | 130 | Mid Brightness |
| Dark Closet | < 500 | > 220 | **Full Brightness** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `(4095 - lightLevel) / 16` | Subtracts the reading from 4095 to invert the range, then divides by 16 to fit 8-bit PWM limits (0–255). |

## Hardware & Safety Concept: Sensor Filtering
Real ambient light levels fluctuate constantly due to shadows, AC light flicker (50Hz/60Hz), and sensor noise. To prevent the LED brightness from flickering rapidly in response to these micro-changes, software filters like **Exponential Moving Average (EMA)** are used to smooth out the raw analog readings.

## Try This! (Challenges)
1. **Light Tracker Mode**: Remove the inversion logic (`4095 - lightLevel`) and make the LED get brighter in bright environments.
2. **Double Indicator**: Add a second LED on GP14 that fades in opposite sync (brightens in daylight).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays OFF in darkness | Constraint logical error | Check that your constraint limits correctly bound the `brightness` variable between `0` and `255`. |

## Mode Notes
This basic analog mapping project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [32 - Pico Potentiometer Dimmer](32-pico-potentiometer-dimmer.md)
- [33 - Pico LDR Sensor](33-pico-ldr-sensor.md)
