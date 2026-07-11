# 31 - Potentiometer ADC Read (GPIO Analog pin)

Read analog voltage values from a potentiometer connected to the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs using the Analog-to-Digital Converter (ADC) on the VEGA ARIES v3 board, convert raw digital readings back into voltage values, and display them on the Serial Monitor.

## What You Will Build
A potentiometer connected to pin `ADC0` (`GP26`) is manually adjusted. The board measures the voltage on the input pin, converts it from a raw 12-bit number (0–4095) to a voltage (0–3.3V), and prints the results to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | VCC | 3.3V | Red | Power connection |
| Potentiometer | Output / Signal | ADC0 (GP26) | Yellow | Analog input signal |
| Potentiometer | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the center pin of the potentiometer (the wiper) to the ARIES analog pin. The side pins go to 3.3V and GND; reversing them will simply invert the direction of rotation required to increase the reading.

## Code
```cpp
// Potentiometer ADC Read - VEGA ARIES v3
const int POT_PIN = GP26; // ADC0

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw 12-bit analog value (0 to 4095)
  int rawValue = analogRead(POT_PIN);
  
  // Convert the raw value to voltage (0.0V to 3.3V)
  float voltage = (rawValue * 3.3) / 4095.0;
  
  // Print results to the Serial Monitor
  Serial.print("Raw ADC Value: ");
  Serial.print(rawValue);
  Serial.print(" | Voltage: ");
  Serial.print(voltage);
  Serial.println(" V");
  
  // Sample every 500 milliseconds
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Potentiometer** onto the canvas.
2. Wire the Potentiometer's VCC pin to **3.3V**, the center Wiper pin to **GP26 (ADC0)**, and the GND pin to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the virtual potentiometer slider and check the Serial Monitor output.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw ADC Value: 0 | Voltage: 0.00 V
Raw ADC Value: 2048 | Voltage: 1.65 V
Raw ADC Value: 4095 | Voltage: 3.30 V
```

## Expected Canvas Behavior
* The value displayed in the Serial Monitor updates dynamically as you adjust the potentiometer's knob on the canvas.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int POT_PIN = GP26;` | Defines the potentiometer signal pin. |
| `analogRead(POT_PIN)` | Triggers the ADC and reads the raw 12-bit value (0-4095). |
| `(rawValue * 3.3) / 4095.0` | Math formula converting raw digital steps to an actual voltage value. |

## Hardware & Safety Concept: ADC Resolution and Voltage Limits
* **ADC Resolution**: The THEJAS32 SoC on the VEGA ARIES v3 features a 12-bit Successive Approximation Register (SAR) Analog-to-Digital Converter. This provides $2^{12} = 4096$ discrete voltage steps between 0V and 3.3V, equating to a resolution of roughly 0.8 mV per step.
* **Voltage Limits**: Standard GPIO pins on the THEJAS32 SoC operate on a 3.3V logic level. Applying voltages higher than 3.3V to the ADC pins can destroy the internal analog gates. Always verify the source voltage before connecting it to an ADC channel.

## Try This! (Challenges)
1. **Threshold Indicator**: Add the Red onboard LED (`LED_R`) and write code to turn it ON if the voltage exceeds 2.5V, and OFF otherwise.
2. **Reading Delta Filter**: Modify the code to print values only if the raw ADC reading changes by more than 50 units compared to the previous loop iteration.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Output stuck at 0 or 4095 | Wrong pins connected or open circuit | Verify the wiring matches GP26 (center pin) and power connections are secured |
| Fluctuating readings when stationary | Electrical noise or floating input | Check that the GND line of the sensor is correctly connected to the common ARIES GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - Potentiometer LED Dimmer](32-potentiometer-led-dimmer.md) (Next project)
