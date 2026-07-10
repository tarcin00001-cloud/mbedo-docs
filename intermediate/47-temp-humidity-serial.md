# 47 - Temp Humidity Serial

Measure and print real-time temperature and humidity readings using a DHT22 sensor.

## Goal
Learn how to import and use the standard `DHT.h` library to read complex multi-byte digital data from a DHT22 sensor.

## What You Will Build
The Arduino reads environmental data from the DHT22 sensor and prints the temperature (in Celsius) and relative humidity (in %) to the Terminal every 1.5 seconds.

**Why D2?** Pin D2 communicates with the sensor's single-wire digital data line using bidirectional pulse timing.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (pull-up for data line) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA (Data) | D2 | Bidirectional data line |
| DHT22 Sensor | GND | GND | Ground reference |

> On physical hardware, a **10k ohm resistor** should be wired as a pull-up between the `SDA` data pin and `VCC` to ensure the communication line remains stable.

## Code
```cpp
#include <DHT.h>

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;

// Initialize DHT library
DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  Serial.begin(9600);
  Serial.println("DHT22 Sensor Monitor Ready");
  
  // Start the sensor
  dht.begin();
}

void loop() {
  // Read temperature as Celsius (default)
  float tempC = dht.readTemperature();
  
  // Read relative humidity (%)
  float humidity = dht.readHumidity();
  
  // Check if any readings failed (isnan = Is Not a Number)
  if (isnan(tempC) || isnan(humidity)) {
    Serial.println("Error: Failed to read from DHT22 sensor!");
  } else {
    Serial.print("Temperature: ");
    Serial.print(tempC);
    Serial.print("C | Humidity: ");
    Serial.print(humidity);
    Serial.println("%");
  }
  
  // Real DHT22 sensors need at least 1-2 seconds between readings
  delay(1500); 
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **DHT22 Sensor** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the DHT22 sensor on the canvas, adjust the temperature and humidity sliders, and watch the values print in the Terminal.

## Expected Output

Terminal:
```
DHT22 Sensor Monitor Ready
Temperature: 24.50C | Humidity: 55.20%
Temperature: 25.10C | Humidity: 54.80%
...
```

### Expected Canvas Behavior

| Slider Temperature | Slider Humidity | Terminal Output values |
| --- | --- | --- |
| 24.0 C | 50.0% | "Temperature: 24.00C | Humidity: 50.00%" |
| 35.5 C | 85.0% | "Temperature: 35.50C | Humidity: 85.00%" |

The Terminal updates dynamically to match the sensor properties.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `#include <DHT.h>` | Imports the Adafruit DHT sensor library shim. |
| `DHT dht(DHT_PIN, DHT22)` | Instantiates a sensor controller object named `dht` on pin D2. |
| `dht.readTemperature()` | Reads the temperature value. Returns `NAN` if the read fails. |
| `isnan(tempC)` | A math check returning true if the reading failed, preventing garbage data printout. |

## Hardware & Safety Concept: Sensor Communication
The **DHT22** (and DHT11) uses a custom single-wire serial protocol to transmit data, which is different from standard protocols like I2C or SPI. The sensor sends a 40-bit packet containing relative humidity, temperature, and a checksum to verify transmission integrity. Because the timing of these pulses is measured in microseconds, the sensor library disables interrupts briefly to read data accurately.

## Try This! (Challenges)
1. **Fahrenheit Conversion**: Modify the code to display temperature in Fahrenheit. (Hint: pass `true` to `dht.readTemperature(true)`).
2. **Comfort Alarm**: Print a message "Comfortable environment" to the Terminal if the temperature is between 20C and 25C and humidity is between 40% and 60%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Prints "Error: Failed to read..." | Sensor data wire not on D2 | Verify the sensor SDA pin is connected to Arduino pin D2. |
| Readings don't update | Loop refresh rate too fast | Real DHT22 sensors need a 2-second warm-up. Ensure your loop delay is at least 1500 ms. |

## Mode Notes
The `DHT` library is shimmed in the MbedO interpreted runtime. `dht.begin()`, `dht.readTemperature()`, and `dht.readHumidity()` are supported.

## Related Projects
- [15 - Analog Meter Serial](../beginner/15-analog-meter-serial.md)
- [48 - Temp Alarm](48-temp-alarm.md)
- [49 - Humidity Alert LED](49-humidity-alert-led.md)
