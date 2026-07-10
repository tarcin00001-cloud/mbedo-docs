# 15 - Analog Meter Serial

Display a real-time analog voltage and percentage readout in the Terminal.

## Goal
Learn how to read an analog input pin, convert the raw ADC value (0-1023) into a voltage value (0.0-5.0V) and a percentage (0-100%), and print them cleanly to the Terminal.

## What You Will Build
As you turn the potentiometer knob on the canvas, the Terminal updates every 200 ms with the raw value, the calculated voltage, and the percentage rotation.

**Why A0?** Pin A0 feeds directly into the Arduino's built-in analog multiplexer and ADC, which translates analog voltages into digital numbers.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer | `potentiometer` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer | 1 | 5V | Positive voltage reference |
| Potentiometer | 2 (Wiper) | A0 | Analog signal connection |
| Potentiometer | 3 | GND | Ground reference |

## Code
```cpp
const int POT_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("Analog Meter Ready");
}

void loop() {
  // Read raw ADC value (0 to 1023)
  int rawVal = analogRead(POT_PIN);
  
  // Calculate voltage (0.0V to 5.0V)
  // Float casting ensures decimal math is used
  float voltage = rawVal * (5.0 / 1023.0);
  
  // Calculate percentage (0% to 100%)
  int percent = map(rawVal, 0, 1023, 0, 100);
  
  // Print formatted output to Terminal
  Serial.print("Raw: ");
  Serial.print(rawVal);
  Serial.print(" | Volts: ");
  Serial.print(voltage);
  Serial.print("V | Percent: ");
  Serial.print(percent);
  Serial.println("%");
  
  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **Potentiometer** onto the canvas.
2. Connect Potentiometer **pin 1** to Arduino **5V**, **pin 2 (wiper)** to Arduino **A0**, and **pin 3** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the Potentiometer to adjust the knob and watch the numbers update in the Terminal.

## Expected Output

Terminal:
```
Analog Meter Ready
Raw: 0 | Volts: 0.00V | Percent: 0%
Raw: 512 | Volts: 2.50V | Percent: 50%
Raw: 1023 | Volts: 5.00V | Percent: 100%
...
```

### Expected Canvas Behavior

| Potentiometer Input | Wiper Voltage | Pin A0 Reading | Calculated Float | Printed Percent |
| --- | --- | --- | --- | --- |
| 0% | 0.0V | 0 | 0.00 | 0% |
| 25% | 1.25V | 256 | 1.25 | 25% |
| 50% | 2.50V | 512 | 2.50 | 50% |
| 100% | 5.00V | 1023 | 5.00 | 100% |

The values update in real-time as the slider moves.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `float voltage = rawVal * (5.0 / 1023.0)` | Converts the raw 10-bit integer (0-1023) into its voltage equivalent. Since the Arduino Uno runs on 5V, dividing 5.0 by 1023.0 gives the voltage step per unit of ADC value. |
| `float` | A data type that stores decimal numbers (floating-point numbers). This is essential for printing voltages like `2.50V`. |
| `map(rawVal, 0, 1023, 0, 100)` | Scales the raw reading directly to a percentage rotation range (0 to 100). |

## Hardware & Safety Concept: ADC Resolution
The Arduino Uno uses a 10-bit Analog-to-Digital Converter (ADC). This means it divides the range between 0V and 5V into 1024 steps. The smallest voltage change the Arduino can detect is:
5V / 1023 = 0.00489V (approx. 4.89 mV).
This is the resolution limit of your analog inputs.

## Try This! (Challenges)
1. **Low Voltage Warning**: Add a conditional statement (`if`) that prints a warning message "Low Battery Alert!" when the calculated voltage drops below 1.5V.
2. **Simple Bar Graph**: Write a conditional block that prints `[===  ]` if the percentage is between 40% and 60%, and `[=====]` if above 80%, creating a simple text-based level bar.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage shows only 0.00V or 5.00V | Wiper pin not wired to A0 | Check that the center pin of the potentiometer is wired to A0 (not the outer pins). |
| Values bounce up and down constantly | Floating input | Make sure the outer pins are wired firmly to 5V and GND. If they are loose, electrical noise will cause wild readings. |

## Mode Notes
These patterns (floating-point calculations, basic math, and serial prints) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [13 - Button Counter Print](13-button-counter-print.md)
- [14 - Dial LED Brightness](14-dial-led-brightness.md)
