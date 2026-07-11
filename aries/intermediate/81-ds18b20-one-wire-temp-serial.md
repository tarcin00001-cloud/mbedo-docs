# 81 - DS18B20 One-Wire Temp Sensor Serial Logs

Read temperature values from a high-precision DS18B20 sensor using the OneWire bus protocol and print the logs to the Serial Monitor.

## Goal
Learn how to use Dallas OneWire protocol buses, address devices on a single data pin, and read temperature data in Celsius and Fahrenheit without using loops inside C++ code.

## What You Will Build
A DS18B20 temperature sensor is connected to GPIO 12 of the ARIES v3 board with a 4.7 kΩ pull-up resistor. The board queries the sensor over the OneWire bus every 1.5 seconds and prints the Celsius and Fahrenheit temperature readings to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| 4.7 kΩ Resistor (pull-up) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensor | VCC (Red) | 3V3 | Red | Power |
| DS18B20 Sensor | GND (Black) | GND | Black | Ground reference |
| DS18B20 Sensor | DATA (Yellow) | GPIO 12 | Yellow | OneWire data bus |
| Pull-up Resistor | VCC to DATA | — | — | Place 4.7k resistor between VCC and DATA pins |

> **Wiring tip:** The DS18B20 requires an external **4.7 kΩ pull-up resistor** (values from 2.2k to 10k are acceptable) between VCC (3.3V) and the DATA line. Without this resistor, the OneWire bus cannot be pulled high, and the ARIES board will fail to detect the sensor (reading -127 °C).

## Code
```cpp
// DS18B20 One-Wire Temperature Logs
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONEWIRE_BUS = 12;

OneWire oneWire(ONEWIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(115200);
  Serial.println("Initializing OneWire bus...");
  
  sensors.begin();
  
  // Count devices on the bus
  int deviceCount = sensors.getDeviceCount();
  Serial.print("Found ");
  Serial.print(deviceCount);
  Serial.println(" DS18B20 sensor(s).");
}

void loop() {
  // Send command to perform temperature conversion
  sensors.requestTemperatures(); 
  
  // Read temperature from the first device (index 0)
  float tempC = sensors.getTempCByIndex(0);
  
  // -127°C is returned by the library if a read failure occurs
  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: Could not read temperature data (Sensor disconnected).");
  } else {
    float tempF = DallasTemperature::toFahrenheit(tempC);
    
    Serial.print("Temp: ");
    Serial.print(tempC, 2);
    Serial.print(" *C (");
    Serial.print(tempF, 2);
    Serial.println(" *F)");
  }
  
  delay(1500); // Wait 1.5 seconds between reads
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **DS18B20 Sensor** components onto the canvas.
2. Wire the sensor: **DATA** to **GPIO 12**, **VCC** to **3V3**, and **GND** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Adjust the temperature slider on the DS18B20 widget and watch the output print to the Serial Monitor.

## Expected Output
Serial Monitor:
```
Initializing OneWire bus...
Found 1 DS18B20 sensor(s).
Temp: 24.50 *C (76.10 *F)
Temp: 27.25 *C (81.05 *F)
```

## Expected Canvas Behavior
* The Serial Monitor lists the number of sensors detected at startup.
* Readings print every 1.5 seconds.
* Adjusting the slider on the DS18B20 component instantly changes the printed values on the next loop cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `OneWire oneWire(12)` | Registers the OneWire protocol driver on GPIO pin 12. |
| `DallasTemperature sensors(...)` | Passes the OneWire instance to the Dallas temperature sensor library. |
| `sensors.requestTemperatures()` | Sends a global broadcast command telling all DS18B20 sensors on the bus to measure temperature. |
| `sensors.getTempCByIndex(0)` | Reads the converted temperature in Celsius from the first sensor on the bus. |
| `DEVICE_DISCONNECTED_C` | A constant (-127) representing a failed read/checksum error. |

## Hardware & Safety Concept
* **Maxim Dallas OneWire Bus**: The Dallas OneWire protocol allows a master device (ARIES v3) to communicate with one or more slave devices using a single data wire and a common ground. Each DS18B20 sensor has a unique, factory-programmed 64-bit ROM address. Because of this, multiple sensors can be hooked up in parallel to the exact same pin (GPIO 12), and the master can address each individually. The water-resistant probe version is ideal for wet environments like aquarium monitors or soil probes.

## Try This! (Challenges)
1. **Alert Trigger**: Print a warning "FREEZE ALERT" if the temperature falls below 5.0 °C.
2. **Temperature Difference**: Connect a second DS18B20 to GPIO 12 and print the temperature difference between the two probes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Readings always show -127.00 | Missing pull-up resistor or bad connection | Add a 4.7 kΩ pull-up resistor between VCC and DATA pins. |
| Code halts at setup | Sensor address conflict | Ensure each sensor on the same bus pin has a unique ID. |
| Temperature reads 85.00 | Read request sent too quickly after power-up | 85 °C is the default power-on register value. Add a delay in `setup()` before the first read. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [82 - DS18B20 Temperature Alarm](82-ds18b20-temperature-alarm.md)
- [83 - DS18B20 Multi-Sensor LCD HUD](83-ds18b20-multi-sensor-lcd-hud.md)
- [71 - DHT22 Temp & Humidity Serial Logs](71-dht22-temp-humidity-serial.md)
