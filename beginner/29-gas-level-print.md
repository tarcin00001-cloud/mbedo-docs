# 29 - Gas Level Print

Measure and display gas/smoke concentration using an MQ-2 Gas Sensor.

## Goal
Learn how to read an analog gas concentration value from an MQ-2 sensor, categorize air quality levels, and output results to the Terminal.

## What You Will Build
As you adjust the gas concentration slider on the MQ-2 sensor, the Terminal prints the raw ADC value and a human-readable air quality category (Good, Moderate, Poor, or Danger) every 500 ms.

**Why A0?** The MQ-2 sensor output voltage changes continuously based on gas concentration, requiring connection to analog pin A0.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Power supply (5V) |
| MQ-2 Sensor | AOUT (Analog Out) | A0 | Analog signal connection |
| MQ-2 Sensor | GND | GND | Ground reference |

> **Note:** The `DOUT` (Digital Out) pin is not used in this beginner project. It is used for digital threshold interrupts in intermediate projects.

## Code
```cpp
const int GAS_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("MQ-2 Gas Monitor Ready");
  Serial.println("Warming up sensor...");
  delay(2000); // Allow sensor simulation to stabilize
}

void loop() {
  int gasLevel = analogRead(GAS_PIN);

  Serial.print("Gas Level (Raw): ");
  Serial.print(gasLevel);

  // Categorize concentration readings into bands
  if (gasLevel < 200) {
    Serial.println(" - Air Quality: GOOD");
  } else if (gasLevel < 500) {
    Serial.println(" - Air Quality: MODERATE");
  } else if (gasLevel < 800) {
    Serial.println(" - Air Quality: POOR");
  } else {
    Serial.println(" - Air Quality: DANGER!");
  }

  delay(500); // Refresh 2 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **MQ2 Gas Sensor** onto the canvas.
2. Connect MQ-2 **VCC** to Arduino **5V**, **AOUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the MQ-2 sensor on the canvas, adjust the gas concentration slider, and watch the air quality labels change in the Terminal.

## Expected Output

Terminal:
```
MQ-2 Gas Monitor Ready
Warming up sensor...
Gas Level (Raw): 85 - Air Quality: GOOD
Gas Level (Raw): 340 - Air Quality: MODERATE
Gas Level (Raw): 850 - Air Quality: DANGER!
...
```

### Expected Canvas Behavior

| Gas Concentration | Pin A0 Reading | Air Quality Category |
| --- | --- | --- |
| Clean Air | < 200 | "Air Quality: GOOD" |
| Light Smoke / Gas Leak | 200 - 500 | "Air Quality: MODERATE" |
| Heavy Gas / Smoke | 500 - 800 | "Air Quality: POOR" |
| Extreme concentration | > 800 | "Air Quality: DANGER!" |

Terminal output categories match the slider positions directly.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `delay(2000)` (setup) | Simulates the sensor pre-heating warm-up phase (real sensors need ~20 seconds to stabilize). |
| `if/else if/else` block | Classifies raw ADC levels into four bands to simplify raw numbers for the user. |

## Hardware & Safety Concept: Chemiresistive Gas Sensors
The **MQ-2** sensor uses a tin dioxide (SnO2) sensing layer. In clean air, SnO2 has high electrical resistance. When combustible gases (LPG, propane, methane, carbon monoxide, or smoke) contact the heated sensor layer, the resistance drops. This resistance drop is converted to an analog voltage. The module has an internal heating coil, which is why it runs warm and requires warm-up time to stabilize.

## Try This! (Challenges)
1. **Safety Alarm**: Wire a Buzzer to D8 and code a siren alarm that triggers only when the gas level enters the "DANGER!" zone (above 800).
2. **Peak Level Recorder**: Keep track of the highest level read during the session and print it alongside each reading.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading is stuck at 0 | AOUT pin not connected | Ensure MQ-2 AOUT connects to A0. DOUT only outputs 0 or 1023. |
| Reading is stuck at 1023 | Sensor not grounded or no power | Check VCC connects to 5V and GND connects to GND. |
| Reading doesn't change with slider | Pin configuration mismatch | Confirm `GAS_PIN = A0` matches the physical wire to pin A0. |

## Mode Notes
These patterns (analog reads and multi-branch comparison logic) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [18 - Light Meter](18-light-meter.md)
- [25 - Rain Level Print](25-rain-level-print.md)
- [32 - Gas Alarm](../intermediate/32-gas-alarm.md)
- [33 - Gas LED Alert](../intermediate/33-gas-led-alert.md)
