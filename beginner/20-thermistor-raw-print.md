# 20 - Thermistor Raw Print

Read raw analog values from an NTC Thermistor and print them to the Terminal.

## Goal
Learn how to read an NTC Thermistor as an analog input and understand how physical temperature shifts alter the resistance and output voltage.

## What You Will Build
As you change the temperature slider on the thermistor component, the Terminal prints the updated raw ADC value every 200 ms.

**Why A0?** The thermistor's voltage divider output must connect to an analog input pin (A0) to be parsed by the ADC.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| NTC Thermistor | `ntc_temp` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (required for discrete divider) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Thermistor | VCC | 5V | Power supply (5V) |
| Thermistor | OUT | A0 | Analog signal connection |
| Thermistor | GND | GND | Ground reference |

## Code
```cpp
const int NTC_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("NTC Thermistor Monitor Ready");
}

void loop() {
  // Read raw thermistor value (0 to 1023)
  int rawVal = analogRead(NTC_PIN);
  
  Serial.print("Thermistor Raw Value: ");
  Serial.println(rawVal);
  
  delay(200); // Read 5 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **NTC Thermistor** onto the canvas.
2. Connect Thermistor **VCC** to Arduino **5V**, **OUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the Thermistor on the canvas, adjust the temperature slider, and watch the values change in the Terminal.

## Expected Output

Terminal:
```
NTC Thermistor Monitor Ready
Thermistor Raw Value: 512
Thermistor Raw Value: 420
Thermistor Raw Value: 680
...
```

### Expected Canvas Behavior

| Temperature State | Sensor Resistance | Signal Voltage | Pin A0 Reading |
| --- | --- | --- | --- |
| Cold | High | High (Near 5V) | High (e.g. > 700) |
| Room Temp | Moderate (10k) | Medium (Near 2.5V) | Medium (e.g. 512) |
| Hot | Low | Low (Near 0V) | Low (e.g. < 300) |

The raw ADC value drops as the temperature rises.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `analogRead(NTC_PIN)` | Reads the analog output voltage from the thermistor's onboard voltage divider. |

## Hardware & Safety Concept: Negative Temperature Coefficient (NTC)
"NTC" stands for **Negative Temperature Coefficient**. This means that as temperature *increases*, the thermistor's electrical resistance *decreases*. 
- The module's built-in voltage divider circuit outputs a voltage that is inversely proportional to the temperature.
- Therefore, a higher temperature yields a lower analog voltage, resulting in a lower raw ADC value.

## Try This! (Challenges)
1. **Temperature Trend Indicator**: Add logic to compare the current reading to the previous reading and print whether the temperature is "Getting Hotter" or "Getting Colder".
2. **Freeze Alert**: Add an `if` statement to print a warning message "Freeze Warning!" when the raw value rises above 900 (which corresponds to cold temperatures).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Value stays stuck at 0 or 1023 | Wiring error | Ensure OUT is connected to A0, VCC to 5V, and GND to GND. |
| Values increase when it gets colder | Normal NTC behavior | This is correct for NTC thermistors. To invert the scale in code, you can subtract the reading from 1023 (`1023 - rawVal`). |

## Mode Notes
These patterns (analog input reading and basic serial prints) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
- [41 - Thermostat Relay](../intermediate/41-thermostat-relay.md)
