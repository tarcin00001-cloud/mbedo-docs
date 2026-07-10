# 81 - Pico DS18B20 Serial

Read and log temperature measurements from a DS18B20 waterproof digital sensor using the One-Wire protocol.

## Goal
Learn how to interface digital One-Wire devices, request temperature conversions, and log telemetry.

## What You Will Build
A waterproof temperature probe log:
- **DS18B20 Sensor (GP12)**: Reads liquid/air temperature every 1.5 seconds and logs it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temperature Sensor| `ds18b20` | Yes | Yes (waterproof probe) |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 | VCC | 3.3V | Power supply |
| DS18B20 | DQ (Data) | GP12 | One-Wire data (requires 4.7k pull-up to 3.3V) |
| DS18B20 | GND | GND | Ground return |

## Code
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONE_WIRE_BUS = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(9600);
  sensors.begin();
  Serial.println("DS18B20 Sensor Logger Online");
}

void loop() {
  // Command all sensors on the bus to perform temperature conversion
  sensors.requestTemperatures(); 
  
  // Read temperature by index 0 (first sensor found on bus)
  float tempC = sensors.getTempCByIndex(0);

  // Check if reading is valid (DS18B20 returns -127 on errors)
  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: DS18B20 sensor disconnected!");
  } else {
    Serial.print("Probe Temp: ");
    Serial.print(tempC);
    Serial.println(" C");
  }

  delay(1500);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **DS18B20 Sensor** onto the canvas.
2. Connect DS18B20: **VCC** to **3V3**, **DQ** to **GP12** (with 4.7k pull-up to 3V3), **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature slider on the DS18B20 model and check the Serial console logs.

## Expected Output

Terminal:
```
DS18B20 Sensor Logger Online
Probe Temp: 24.50 C
Probe Temp: 28.10 C
```

## Expected Canvas Behavior
| DS18B20 Temperature Slider | Serial Output Temperature |
| --- | --- |
| Ice water | ~0 C |
| Room Temp | ~24 C |
| Boiling water | ~100 C |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensors.requestTemperatures()` | Broadcasts a pulse telling all DS18B20 devices on the shared bus to start internal analog-to-digital temperature calculations. |
| `sensors.getTempCByIndex(0)` | Requests the calculated temperature value from the sensor matching index 0 on the bus. |

## Hardware & Safety Concept: One-Wire Bus Addressing
The One-Wire protocol allows multiple DS18B20 sensors to connect to the exact same physical GPIO pin. Each sensor has a unique, laser-etched **64-bit ROM address** configured at the factory. The library can address a specific probe by its address code, or search the bus and index them dynamically (e.g. index 0, index 1), allowing you to build multi-sensor grids using just one microcontroller pin.

## Try This! (Challenges)
1. **Fahrenheit Log**: Print temperature in Fahrenheit using `sensors.getTempFByIndex(0)`.
2. **Freeze Alert LED**: Turn ON an external LED on GP15 if the temperature drops below 5°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads static `-127.00 C` | Missing pull-up resistor | Ensure a 4.7k (or 10k) resistor is wired between the data line GP12 and the 3.3V power pin. |

## Mode Notes
This basic sensor project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [71 - Pico DHT Serial](71-pico-dht-serial.md)
- [82 - Pico DS18B20 Alarm](82-pico-ds18b20-alarm.md)
