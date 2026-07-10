# 25 - Rain Level Print

Measure and display exact rain wetness levels using a Rain Sensor's analog output.

## Goal
Learn how to read an analog input from a rain sensor grid, map the raw values to categorize rain intensity, and log the results to the Terminal.

## What You Will Build
As you change the wetness slider of the rain sensor, the Terminal displays the raw analog value and a weather description (Dry, Moderate, or Heavy) every 500 ms.

**Why A0?** The rain sensor grid acts as a variable resistor. The analog voltage output from the divider circuit must connect to A0 to measure exact wetness levels rather than just a simple ON/OFF limit.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Rain Sensor | `rain_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Rain Sensor | VCC | 5V | Power supply (5V) |
| Rain Sensor | AO (Analog Out) | A0 | Analog signal connection |
| Rain Sensor | GND | GND | Ground reference |

## Code
```cpp
const int RAIN_AO_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("Rain Level Monitor Ready");
}

void loop() {
  int rainLevel = analogRead(RAIN_AO_PIN);
  
  Serial.print("Rain Wetness (Raw ADC): ");
  Serial.print(rainLevel);
  
  // Note: On physical sensors, a lower value means more water
  // because water increases conductivity (reduces resistance)
  if (rainLevel < 300) {
    Serial.println(" - Heavy Rain!");
  } else if (rainLevel < 700) {
    Serial.println(" - Moderate Rain");
  } else {
    Serial.println(" - Dry / Light Drizzle");
  }
  
  delay(500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **Rain Sensor** onto the canvas.
2. Connect Rain Sensor **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the Rain Sensor, adjust the "wetness" slider, and watch the values and categories print in the Terminal.

## Expected Output

Terminal:
```
Rain Level Monitor Ready
Rain Wetness (Raw ADC): 1023 - Dry / Light Drizzle
Rain Wetness (Raw ADC): 520 - Moderate Rain
Rain Wetness (Raw ADC): 150 - Heavy Rain!
...
```

### Expected Canvas Behavior

| Rain Sensor State | Output Voltage | Pin A0 Reading | Printed Category |
| --- | --- | --- | --- |
| Dry | Near 5V | > 700 | "Dry / Light Drizzle" |
| Drizzle / Damp | Intermediate | 300 - 700 | "Moderate Rain" |
| Heavy Wetness | Near 0V | < 300 | "Heavy Rain!" |

The printed description changes dynamically as the slider moves.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int rainLevel = analogRead(RAIN_AO_PIN)` | Reads the analog output voltage from the sensor plate. |
| `if (rainLevel < 300)` | Active-low check. A wet plate conducts more electricity, bringing the voltage down toward 0V (0). A dry plate has high resistance, keeping the voltage near 5V (1023). |

## Hardware & Safety Concept: Sensor Degradation (Electrolysis)
In real-world applications, running a continuous DC current through a resistive sensor exposed to rain and moisture causes rapid metal corrosion due to **electrolysis**. To prevent this, professional outdoor stations only power the sensor pin right before taking a reading, and immediately turn it off afterward.

## Try This! (Challenges)
1. **Calibrated Percentage**: Convert the reading so that it shows `0%` when dry and `100%` when fully wet. (Hint: Use `map(rainLevel, 1023, 0, 0, 100)`).
2. **Alert beep**: Add a Buzzer to D8 and sound a short chirp only when the weather changes from Dry to Moderate Rain.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading stays stuck at 1023 even when slider is high | AO pin wired incorrectly | Verify that the `AO` pin is wired to A0. If you wire `DO` instead, it will only register 0 or 1023. |
| Code doesn't compile | Wrong pin naming | Make sure A0 is written as `A0` (capital letter A and number zero). |

## Mode Notes
These patterns (analog reads and multi-branch comparison blocks) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
- [24 - Rain Alert](24-rain-alert.md)
