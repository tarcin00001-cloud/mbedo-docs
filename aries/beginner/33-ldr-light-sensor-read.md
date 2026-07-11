# 33 - LDR Light Sensor Read (GPIO Analog pin)

Read ambient light intensity values using a Light Dependent Resistor (LDR) on the VEGA ARIES v3 board.

## Goal
Understand the electrical properties of a Light Dependent Resistor (LDR), learn how to wire a voltage divider circuit to translate resistance into voltage, and read the light levels using the board's ADC.

## What You Will Build
An LDR is configured in a voltage divider circuit with a 10 kΩ resistor on analog input pin `ADC1` (`GP27`). The board reads the voltage at the junction, maps it to a percentage (0% for total darkness to 100% for full brightness), and prints it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LDR | Pin 1 | 3.3V | Red | Power connection |
| LDR | Pin 2 | ADC1 (GP27) | Yellow | Analog input signal (at voltage divider midpoint) |
| 10 kΩ Resistor | Pin 1 | ADC1 (GP27) | Yellow | Connected to LDR Pin 2 |
| 10 kΩ Resistor | Pin 2 | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 10 kΩ pull-down resistor in series with the LDR to form a reliable voltage divider. This configuration yields higher voltage values in brighter environments.

## Code
```cpp
// LDR Light Sensor Read - VEGA ARIES v3
const int LDR_PIN = GP27; // ADC1

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw analog voltage at the divider midpoint
  int rawValue = analogRead(LDR_PIN);
  
  // Map raw value (0-4095) to percentage (0% dark to 100% bright)
  int lightPercentage = map(rawValue, 0, 4095, 0, 100);
  
  // Print values to the Serial Monitor
  Serial.print("Raw LDR Value: ");
  Serial.print(rawValue);
  Serial.print(" | Light Level: ");
  Serial.print(lightPercentage);
  Serial.println("%");
  
  // Read light levels every 1 second
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, an **LDR**, a **10 kΩ Resistor**, and a **Breadboard** onto the canvas.
2. Wire the LDR and resistor in series between **3.3V** and **GND**.
3. Connect the junction between the LDR and resistor to **GP27 (ADC1)**.
4. Paste the code into the editor.
5. Click **Run**.
6. Adjust the ambient light slider on the LDR widget to see the change in the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw LDR Value: 1024 | Light Level: 25%
Raw LDR Value: 3072 | Light Level: 75%
```

## Expected Canvas Behavior
* The serial terminal prints the updated light levels dynamically when the slider on the virtual LDR is dragged.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int LDR_PIN = GP27;` | Defines GP27 (ADC1) as the analog LDR signal input pin. |
| `analogRead(LDR_PIN)` | Reads the 12-bit ADC voltage at the divider midpoint. |
| `map(rawValue, 0, 4095, 0, 100)` | Translates the raw reading (0–4095) to a readable percentage (0–100%). |

## Hardware & Safety Concept: Voltage Dividers and Resistance Variation
* **Voltage Dividers**: An LDR's resistance changes with light, but microcontrollers measure voltage, not resistance. Placing the LDR in series with a fixed resistor forms a voltage divider, converting resistance changes into measurable voltage variations.
* **Resistance Variation**: In bright light, an LDR's resistance drops significantly (down to a few hundred ohms), causing the output voltage to pull up close to 3.3V. In darkness, its resistance rises (up to several mega-ohms), pulling the output down to GND.

## Try This! (Challenges)
1. **Calibration Offset**: Check the minimum and maximum readings of your LDR under local lighting conditions, and change the map function's input range to match those limits (e.g. `map(rawValue, 200, 3800, 0, 100)`).
2. **LED Darkness Alert**: Add the onboard Green LED (`LED_G`) and turn it ON only when the light level is above 70% (sunny indicator).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings do not change when light varies | Missing resistor or incorrect divider connection | Ensure the resistor is connected to GND and that the sensor wire goes to the junction between the LDR and the resistor |
| Readings decrease when light increases | Inverted divider setup | If the LDR is connected to GND and the resistor to 3.3V, the voltage will drop as light levels rise. Swap the sensor and resistor positions |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - Potentiometer LED Dimmer](32-potentiometer-led-dimmer.md) (Previous project)
- [34 - LDR Automatic Dark Detector](34-ldr-automatic-dark-detector.md) (Next project)
