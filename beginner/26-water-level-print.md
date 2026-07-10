# 26 - Water Level Print

Measure and display liquid depth using a Water Level Sensor.

## Goal
Learn how to read an analog water level sensor, scale the raw ADC reading into a depth percentage (0-100%), and output the results to the Terminal.

## What You Will Build
As you change the water level slider on the sensor component, the Terminal prints the raw reading and the calculated depth percentage every 300 ms.

**Why A0?** Pin A0 measures the varying voltage output from the water level sensor's parallel traces.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Water Level Sensor | `water_level_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Water Level Sensor | VCC | 5V | Power supply (5V) |
| Water Level Sensor | SIG (Signal) | A0 | Analog signal connection |
| Water Level Sensor | GND | GND | Ground reference |

## Code
```cpp
const int WATER_PIN = A0;

void setup() {
  Serial.begin(9600);
  Serial.println("Water Level Monitor Ready");
}

void loop() {
  int rawVal = analogRead(WATER_PIN);
  
  // Convert 0-1023 analog range to 0-100% depth
  int percentage = map(rawVal, 0, 1023, 0, 100);
  
  Serial.print("Depth (Raw): ");
  Serial.print(rawVal);
  Serial.print(" | Level: ");
  Serial.print(percentage);
  Serial.println("%");
  
  delay(300); // Poll 3 times per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **Water Level Sensor** onto the canvas.
2. Connect Water Level Sensor **VCC** to Arduino **5V**, **SIG** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the Water Level Sensor on the canvas, adjust the level slider, and watch the percentage change.

## Expected Output

Terminal:
```
Water Level Monitor Ready
Depth (Raw): 0 | Level: 0%
Depth (Raw): 512 | Level: 50%
Depth (Raw): 1023 | Level: 100%
...
```

### Expected Canvas Behavior

| Water Depth | Sensor Contact | Output Voltage | Pin A0 Reading | Printed Level |
| --- | --- | --- | --- | --- |
| Dry | None | 0V | 0 | 0% |
| Half-Submerged | Partial | 2.5V | 512 | 50% |
| Fully Submerged | Max | 5V | 1023 | 100% |

The printed percentage matches the slider height directly.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int percentage = map(rawVal, 0, 1023, 0, 100)` | Uses `map()` to translate the full ADC range (0-1023) directly to a percentage scale (0-100%). |
| `analogRead(WATER_PIN)` | Reads the analog output voltage. Unlike the rain sensor, the water level sensor's raw reading *increases* as the depth increases, because more submerged surface area increases conductivity. |

## Hardware & Safety Concept: Resistive Water Depth Sensors
The **water level sensor** has a series of parallel exposed copper traces. These traces act as a variable resistor. When submerged, the water bridges the positive traces (connected to VCC) and the signal traces. The deeper the sensor is submerged, the more traces are bridged by water, lowering the resistance and raising the output voltage on the `SIG` pin.

## Try This! (Challenges)
1. **Low Level Alert**: Add a Buzzer to D8 and code a slow, repeating warning beep (`tone(8, 500, 200); delay(1000);`) if the water level percentage drops below 10% (simulating an empty tank warning).
2. **Overflow Indicator**: Add an LED to D13 and make it turn ON if the water level exceeds 90% (simulating a high water warning).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reading stays at 0 | SIG pin wired incorrectly | Verify that the `SIG` pin is wired to A0. |
| Value fluctuates randomly when dry | Floating ground | Ensure the sensor's GND pin is connected securely to one of the Arduino's GND pins. |

## Mode Notes
These patterns (analog reads, `map()` scaling, and variable-based printing) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [15 - Analog Meter Serial](15-analog-meter-serial.md)
- [40 - Water Pump Control](../intermediate/40-water-pump-control.md)
