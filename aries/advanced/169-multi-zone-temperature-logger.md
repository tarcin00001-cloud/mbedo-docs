# 169 - Multi-zone Temperature Logger

Read temperature from three DS18B20 sensors sharing a single OneWire bus on GPIO 12, identify each sensor by its unique ROM address, and log labelled zone temperatures (Zone A, Zone B, Zone C) to the Serial Monitor every 5 seconds.

## Goal
Learn how multiple DS18B20 sensors can co-exist on a single data wire using their unique 64-bit ROM addresses, how to request and read temperature from each sensor individually by index, and how to format a structured multi-zone temperature log without using arrays.

## What You Will Build
Three DS18B20 waterproof temperature probes share a single OneWire bus on GPIO 12 (with a 4.7 kΩ pull-up to 3V3). The DallasTemperature library enumerates sensors by their bus index (0, 1, 2). The board requests a temperature conversion from all sensors simultaneously, waits for the conversion to complete, then reads each sensor's temperature by index and prints a formatted 3-zone log. A running minimum and maximum is maintained for each zone using six float state variables.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS18B20 Waterproof Temperature Sensor × 3 | `ds18b20` | Yes | Yes |
| 4.7 kΩ Pull-up Resistor | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Zone A | VCC (red wire) | 3V3 | Red | Sensor power |
| DS18B20 Zone A | DATA (yellow wire) | GPIO 12 | Yellow | OneWire bus — all DATA wires join here |
| DS18B20 Zone A | GND (black wire) | GND | Black | Common ground |
| DS18B20 Zone B | VCC (red wire) | 3V3 | Red | Sensor power |
| DS18B20 Zone B | DATA (yellow wire) | GPIO 12 | Yellow | Same OneWire bus as Zone A |
| DS18B20 Zone B | GND (black wire) | GND | Black | Common ground |
| DS18B20 Zone C | VCC (red wire) | 3V3 | Red | Sensor power |
| DS18B20 Zone C | DATA (yellow wire) | GPIO 12 | Yellow | Same OneWire bus as Zone A and B |
| DS18B20 Zone C | GND (black wire) | GND | Black | Common ground |
| Pull-up Resistor | 3V3 to GPIO 12 | — | — | Single 4.7 kΩ resistor for entire bus |

> **Wiring tip:** All three DS18B20 DATA wires connect to the same physical wire that runs to GPIO 12. A single 4.7 kΩ pull-up resistor from this shared wire to 3V3 is all that is needed — do not add individual pull-ups per sensor as this reduces the effective pull-up resistance and may cause bus errors. Keep the total bus length under 10 metres for reliable communication. Sensor index assignments (0, 1, 2) are assigned by the library in the order sensors are discovered on the bus; these may change if sensors are added or removed, so note which physical sensor corresponds to which index during initial setup.

## Code
```cpp
// Multi-zone Temperature Logger
// Three DS18B20 sensors on OneWire bus: GPIO 12

#include <OneWire.h>
#include <DallasTemperature.h>

#define OW_PIN  12

OneWire           oneWire(OW_PIN);
DallasTemperature sensors(&oneWire);

// Current temperature readings for each zone
float tempA = 0.0f;
float tempB = 0.0f;
float tempC = 0.0f;

// Running min/max for each zone (initialised to extremes)
float minA =  999.0f;  float maxA = -999.0f;
float minB =  999.0f;  float maxB = -999.0f;
float minC =  999.0f;  float maxC = -999.0f;

// Sensor count detected on bus
int sensorCount = 0;
long logCount   = 0L;

void setup() {
  Serial.begin(115200);
  sensors.begin();
  sensorCount = sensors.getDeviceCount();

  Serial.println("=== Multi-zone Temperature Logger ===");
  Serial.print("Sensors found on OneWire bus: ");
  Serial.println(sensorCount);

  if (sensorCount < 3) {
    Serial.println("WARNING: Expected 3 sensors. Check wiring and pull-up.");
  }

  Serial.println("Zone A | Zone B | Zone C (°C)");
  Serial.println("----------------------------");
  delay(1000);
}

void loop() {
  // Request temperature conversion from all sensors simultaneously
  sensors.requestTemperatures();

  // Read each sensor by bus index
  tempA = sensors.getTempCByIndex(0);
  tempB = sensors.getTempCByIndex(1);
  tempC = sensors.getTempCByIndex(2);

  logCount++;

  // Update zone A min/max
  if (tempA > -100.0f && tempA < 125.0f) {
    if (tempA < minA) minA = tempA;
    if (tempA > maxA) maxA = tempA;
  }

  // Update zone B min/max
  if (tempB > -100.0f && tempB < 125.0f) {
    if (tempB < minB) minB = tempB;
    if (tempB > maxB) maxB = tempB;
  }

  // Update zone C min/max
  if (tempC > -100.0f && tempC < 125.0f) {
    if (tempC < minC) minC = tempC;
    if (tempC > maxC) maxC = tempC;
  }

  // Print log line
  Serial.print("Log #");
  Serial.print(logCount);
  Serial.print("  |  A: ");
  Serial.print(tempA, 1);
  Serial.print(" C  |  B: ");
  Serial.print(tempB, 1);
  Serial.print(" C  |  C: ");
  Serial.print(tempC, 1);
  Serial.println(" C");

  // Print min/max summary every 5 logs
  if (logCount % 5 == 0) {
    Serial.println("--- Min/Max Summary ---");
    Serial.print("Zone A: Min="); Serial.print(minA, 1);
    Serial.print(" C  Max="); Serial.print(maxA, 1); Serial.println(" C");
    Serial.print("Zone B: Min="); Serial.print(minB, 1);
    Serial.print(" C  Max="); Serial.print(maxB, 1); Serial.println(" C");
    Serial.print("Zone C: Min="); Serial.print(minC, 1);
    Serial.print(" C  Max="); Serial.print(maxC, 1); Serial.println(" C");
    Serial.println("-----------------------");
  }

  delay(5000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Drag three **DS18B20** components onto the canvas.
3. Connect all three **DATA** pins to **GPIO 12** (the same bus node).
4. Connect all three **VCC** pins to **3V3** and all three **GND** pins to **GND**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** from the simulation dropdown.
7. Click **Run**.
8. Adjust each DS18B20 temperature slider independently; the Serial Monitor will print a log with all three zone temperatures every 5 seconds.
9. After 5 log entries, a min/max summary for all three zones will be printed.

## Expected Output
Serial Monitor:
```
=== Multi-zone Temperature Logger ===
Sensors found on OneWire bus: 3
Zone A | Zone B | Zone C (°C)
----------------------------
Log #1  |  A: 22.5 C  |  B: 28.3 C  |  C: 19.0 C
Log #2  |  A: 22.6 C  |  B: 29.1 C  |  C: 19.2 C
Log #3  |  A: 23.0 C  |  B: 27.8 C  |  C: 18.9 C
Log #4  |  A: 23.4 C  |  B: 27.5 C  |  C: 19.5 C
Log #5  |  A: 23.1 C  |  B: 28.0 C  |  C: 19.3 C
--- Min/Max Summary ---
Zone A: Min=22.5 C  Max=23.4 C
Zone B: Min=27.5 C  Max=29.1 C
Zone C: Min=18.9 C  Max=19.5 C
-----------------------
```

## Expected Canvas Behavior
* All three DS18B20 components are visible on the canvas, each with an independent temperature slider.
* The Serial Monitor updates every 5 seconds with readings from all three zones.
* Each sensor's slider change is reflected in the next log line.
* Every fifth log entry triggers a min/max summary table in the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <OneWire.h>` | Imports the OneWire protocol library. |
| `#include <DallasTemperature.h>` | Imports the DS18B20 sensor driver that wraps OneWire. |
| `sensors.begin()` | Initialises the OneWire bus and discovers all DS18B20 sensors. |
| `sensors.getDeviceCount()` | Returns the number of DS18B20 sensors found on the bus. |
| `sensors.requestTemperatures()` | Broadcasts the temperature conversion command to all sensors simultaneously. |
| `sensors.getTempCByIndex(0)` | Reads the temperature from the first sensor found on the bus. |
| `if (tempA > -100.0f && tempA < 125.0f)` | Range-checks the reading before updating min/max; DS18B20 returns -127 on error. |
| `if (tempA < minA) minA = tempA` | Updates the running minimum without any array or loop. |
| `if (logCount % 5 == 0)` | Prints the min/max summary every 5th log entry. |

## Hardware & Safety Concept
* **OneWire Multi-drop Bus**: The DS18B20 uses a single-wire protocol where each sensor has a unique 64-bit ROM address burned in at the factory. The DallasTemperature library uses these addresses to communicate with each sensor individually over the shared wire. `getTempCByIndex(n)` retrieves the last converted temperature from the sensor at enumeration index n. The order of indices corresponds to the order in which sensors were discovered during `begin()`, which is determined by their ROM addresses.
* **Simultaneous Conversion**: Calling `sensors.requestTemperatures()` once sends a broadcast convert command that all sensors on the bus respond to in parallel. After waiting for the conversion time (750 ms for 12-bit resolution), each sensor's value is stored in its scratchpad register ready to be read. This is more efficient than issuing individual conversion commands per sensor.
* **Error Value Detection**: The DS18B20 returns -127 °C when it fails to communicate or is disconnected. The validity check `tempA > -100.0f && tempA < 125.0f` filters out this error value before using it in min/max calculations or printing.

## Try This! (Challenges)
1. **Zone Alert**: Compare all three zone temperatures and print "HOTTEST ZONE: A" (or B or C) in the summary block, identifying which zone is currently warmest. Use three `if/else` comparisons on the current readings without any array operations.
2. **Temperature Differential Alarm**: Compute the difference between the highest and lowest zone temperatures (`maxTemp - minTemp`). If this differential exceeds 10 °C, toggle the buzzer on GPIO 14 briefly to indicate a dangerous temperature gradient between zones.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor count shows 0 | No pull-up resistor or DATA not connected to GPIO 12 | Verify all three DATA wires connect to GPIO 12; confirm 4.7 kΩ pull-up to 3V3. |
| One zone reads -127 °C | That DS18B20 has a loose connection or is damaged | Re-seat the sensor connection; replace the sensor if the problem persists. |
| All three zones read the same value | Sensors not uniquely indexed — bus discovery issue | Power-cycle the board to force `begin()` to re-enumerate; confirm each DS18B20 is a different physical sensor. |
| Min/max summary not printed | `logCount % 5` condition never triggers | Ensure `logCount++` is incremented before the modulo check and that `delay(5000)` does not reset the counter. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [163 - Water Quality Station](163-water-quality-station.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
- [170 - NeoPixel Color Spectrum Visualizer](170-neopixel-color-spectrum-visualizer.md)
