# 71 - Pico DHT Serial

Measure and log ambient temperature and humidity readings from a DHT22 sensor to the Serial Monitor.

## Goal
Learn how to interface single-wire digital sensors, parse digital data packets, and log environmental telemetry.

## What You Will Build
An environmental tracker:
- **DHT22 Sensor (GP12)**: Reads current temperature (°C) and humidity (%) every 2 seconds and prints the values to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 3.3V | Power supply |
| DHT22 | SDA (Data Out) | GP12 | Data signal (requires 10k pull-up to VCC) |
| DHT22 | GND | GND | Ground reference |

## Code
```cpp
#include <DHT.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

void setup() {
  Serial.begin(9600);
  dht.begin();
  Serial.println("DHT22 Sensor Monitor Online");
}

void loop() {
  // Read temperature as Celsius
  float t = dht.readTemperature();
  // Read humidity percentage
  float h = dht.readHumidity();

  // Validate sensor communication
  if (isnan(t) || isnan(h)) {
    Serial.println("Error: Failed to read from DHT sensor!");
  } else {
    Serial.print("Temp: ");
    Serial.print(t);
    Serial.print(" C | Humid: ");
    Serial.print(h);
    Serial.println(" %");
  }

  delay(2000); // DHT22 requires 2 seconds between samples
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico** and **DHT22 Sensor** onto the canvas.
2. Connect DHT22: **VCC** to **3V3**, **SDA** to **GP12** (with 10k resistor to 3V3), **GND** to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature slider on the DHT22 model and view the Serial logs.

## Expected Output

Terminal:
```
DHT22 Sensor Monitor Online
Temp: 24.50 C | Humid: 55.20 %
Temp: 26.10 C | Humid: 52.80 %
```

## Expected Canvas Behavior
| DHT22 Slider Settings | Serial Temp output | Serial Humid output |
| --- | --- | --- |
| Normal Room | ~24.0 C | ~50% |
| High Heat | ~40.0 C | ~80% |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `isnan(t)` | Standard check utility verifying if the float variable is "Not a Number", indicating a transmission failure. |

## Hardware & Safety Concept: Single-wire Pull-ups
The DHT22 uses a custom single-wire bus protocol where both the microcontroller and the sensor send data sequentially on the same wire. When idle, the bus line must be held high. A **10k-ohm pull-up resistor** connected between the data line and VCC keeps the bus stable. Without this resistor, the line stays floating and data bits get corrupted.

## Try This! (Challenges)
1. **Fahrenheit Scale**: Convert the temperature reading to Fahrenheit using `dht.readTemperature(true)` and print the Fahrenheit reading.
2. **Heat Warning**: Turn ON the built-in LED on GP25 if the temperature rises above 30.0°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial displays "Failed to read from DHT sensor!" | Missing pull-up resistor | Ensure a 10k resistor is wired between GP12 and the Pico's 3.3V pin. |

## Mode Notes
This basic sensor project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [72 - Pico DHT Alarm](72-pico-dht-alarm.md)
- [73 - Pico DHT LCD](73-pico-dht-lcd.md)
