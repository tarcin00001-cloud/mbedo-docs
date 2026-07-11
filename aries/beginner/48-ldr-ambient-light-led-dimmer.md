# 48 - LDR Ambient Light LED Dimmer (GPIO PWM)

Build an automatic brightness compensation system that fades an LED brighter as ambient light decreases.

## Goal
Learn how to map an analog sensor input range inversely to a Pulse Width Modulation (PWM) duty cycle, creating an automatic brightness control system (similar to a smartphone auto-brightness display).

## What You Will Build
An LDR is configured in a voltage divider circuit on analog input pin `ADC1` (`GP27`), and an external LED is connected to `GPIO 15`. The board reads ambient light levels and inversely maps them: as the environment gets darker, the LED fades brighter; as the environment gets lighter, the LED dims.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Red | Power connection |
| LDR | Pin 2 | ADC1 (GP27) | Yellow | Analog input signal (at voltage divider midpoint) |
| 10 kΩ Resistor | Pin 1 | ADC1 (GP27) | Yellow | Connected to LDR Pin 2 |
| 10 kΩ Resistor | Pin 2 | GND | Black | Ground connection |
| LED | Anode (+) | GPIO 15 | Red | PWM control signal (via resistor) |
| LED | Cathode (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 10 kΩ resistor to pull down the LDR signal on GP27 (ADC1), and a 220 Ω resistor in series with the LED on GPIO 15 to ensure stable, safe current levels for both components.

## Code
```cpp
// LDR Ambient Light LED Dimmer - VEGA ARIES v3
const int LDR_PIN = GP27; // ADC1
const int LED_PIN = 15;   // Warning LED

void setup() {
  // Configure the LED pin as output
  pinMode(LED_PIN, OUTPUT);
}

void loop() {
  // Read the raw analog voltage at the divider midpoint
  int lightVal = analogRead(LDR_PIN);
  
  // Map LDR: 0 (dark) to 255 (full LED brightness), 4095 (bright) to 0 (LED OFF)
  // This inverse mapping creates the automatic compensation effect
  int ledBrightness = map(lightVal, 0, 4095, 255, 0);
  
  // Clamp variables to prevent values outside standard 8-bit PWM bounds
  if (ledBrightness < 0) {
    ledBrightness = 0;
  }
  if (ledBrightness > 255) {
    ledBrightness = 255;
  }
  
  // Write the PWM duty cycle to the LED
  analogWrite(LED_PIN, ledBrightness);
  
  // Wait 50ms before adjusting level
  delay(50);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LDR**, a **10 kΩ Resistor**, an **LED**, and a **220 Ω Resistor** onto the canvas.
2. Wire the LDR voltage divider to **GP27 (ADC1)**.
3. Wire the LED's anode to **GPIO 15** through the resistor and the cathode to **GND**.
4. Paste the code into the editor.
5. Click **Run**.
6. Slide the ambient light slider on the LDR widget. Lower light settings will cause the LED to brighten.

## Expected Output
Serial Monitor:
```
System Initialized.
```

## Expected Canvas Behavior
* The external LED fades brighter as the LDR slider is moved towards the dark setting, and dims to off as the LDR is moved towards the bright setting.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int LDR_PIN = GP27;` | Configures GP27 (ADC1) as the analog LDR signal input pin. |
| `map(lightVal, 0, 4095, 255, 0)` | Maps the LDR reading (0 to 4095) inversely to a PWM duty cycle (255 down to 0). |
| `analogWrite(LED_PIN, ledBrightness)` | Writes the inverse duty cycle to the LED pin using PWM. |

## Hardware & Safety Concept: Inverse Relationship Mapping and Power Efficiency
* **Inverse Relationship Mapping**: In auto-brightness systems, output brightness must be inversely proportional to ambient light. Software scaling easily accommodates this by mapping the minimum sensor range to maximum actuator output, and vice versa.
* **Power Efficiency**: Using PWM instead of turning the LED fully on and off saves significant energy, which is a major factor in battery-powered devices like smart streetlights or mobile displays.

## Try This! (Challenges)
1. **Auto-Brightness Smoothing**: The human eye is sensitive to sudden changes in light. Implement a smoothing filter (running average or small step increments/decrements) to slowly fade the LED brightness to its new target rather than jumping instantly.
2. **Daylight Threshold Cutoff**: Modify the code to keep the LED completely OFF if the ambient light percentage is above 60%, regardless of minor sensor oscillations.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED gets brighter when light increases | Mapping function is not inverted | Verify that the mapping parameters are `map(lightVal, 0, 4095, 255, 0)` and not `0, 255` |
| LED stays at a constant brightness | Using digital output or wrong pin | Check that the code calls `analogWrite()` and that the LED is connected to GPIO 15, which supports PWM |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [47 - Joysticks Y-Axis read](47-joysticks-y-axis-read.md) (Previous project)
- [49 - Sound sensor threshold alarm](49-sound-sensor-threshold-alarm.md) (Next project)
