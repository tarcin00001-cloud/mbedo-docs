# 64 - DS18B20 Temp Print

Read and display high-precision digital temperature values from a DS18B20 One-Wire sensor.

## Goal
Learn how to use the `OneWire.h` and `DallasTemperature.h` libraries to communicate with a DS18B20 digital thermometer.

## What You Will Build
The Arduino polls the DS18B20 sensor using the One-Wire protocol and prints the temperature (in Celsius) to the Terminal every 1 second.

**Why D2?** Pin D2 is configured as the One-Wire bus signal pin, allowing the Arduino to address and request readings from multiple devices on the same single wire.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DS18B20 Temp Sensor | `ds18b20` | Yes | Yes |
| 4.7k ohm resistor | `resistor` | Optional | Yes (pull-up for DQ data line) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 Sensor | VCC | 5V | Power supply (5V) |
| DS18B20 Sensor | DQ (Data) | D2 | One-Wire data connection |
| DS18B20 Sensor | GND | GND | Ground reference |

> On physical hardware, a **4.7k ohm resistor** must be wired as a pull-up between the `DQ` data pin and `VCC` to ensure stable signal transitions on the One-Wire bus.

## Code
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONE_WIRE_BUS = 2;

// Set up a oneWire instance to communicate with any OneWire device
OneWire oneWire(ONE_WIRE_BUS);

// Pass our oneWire reference to Dallas Temperature library
DallasTemperature sensors(&oneWire);

void setup() {
  Serial.begin(9600);
  Serial.println("DS18B20 Temperature Station Ready");
  
  // Start the library
  sensors.begin();
}

void loop() {
  // Send command to all sensors on the bus to take readings
  sensors.requestTemperatures();
  
  // Fetch temperature from the first sensor (index 0)
  float tempC = sensors.getTempCByIndex(0);
  
  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: DS18B20 sensor not found!");
  } else {
    Serial.print("Temperature: ");
    Serial.print(tempC);
    Serial.println(" C");
  }
  
  delay(1000); // 1 Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** and **DS18B20 Temperature Sensor** onto the canvas.
2. Connect DS18B20 **VCC** to Arduino **5V**, **DQ** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Paste the code into the editor.
4. Use the default Arduino interpreted runtime.
5. Click **Run**.
6. Open the **Terminal** tab in the bottom dock.
7. Double-click the DS18B20 sensor, adjust the temperature slider, and watch the values update in the Terminal.

## Expected Output

Terminal:
```
DS18B20 Temperature Station Ready
Temperature: 24.50 C
Temperature: 35.10 C
...
```

### Expected Canvas Behavior

| Slider Temperature Input | Pin D2 Activity | Terminal Output |
| --- | --- | --- |
| 22.5C | One-Wire data pulses | "Temperature: 22.50 C" |
| 45.0C | One-Wire data pulses | "Temperature: 45.00 C" |

The Terminal printout updates to match the sensor properties in real-time.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `OneWire oneWire(ONE_WIRE_BUS)` | Establishes the low-level One-Wire communication protocol on pin D2. |
| `DallasTemperature sensors(&oneWire)` | Creates the higher-level driver object using the OneWire protocol helper. |
| `sensors.requestTemperatures()` | Broadcasts a global command instructing all sensors on the bus to perform temperature conversions. |
| `sensors.getTempCByIndex(0)` | Fetches the calculated Celsius value from the first sensor discovered on the bus. |

## Hardware & Safety Concept: One-Wire Protocol
The **One-Wire** protocol (developed by Dallas Semiconductor) allows a host controller to communicate with multiple slave devices using only a single data line (plus ground).
- Every DS18B20 sensor has a unique, factory-programmed **64-bit registration number** burned into its ROM.
- The Arduino addresses each device individually by broadcasting its registration code. This allows you to wire dozens of digital thermometers in parallel to the exact same Arduino pin (D2), saving I/O pins for other uses.

## Try This! (Challenges)
1. **Fahrenheit Display**: Map the output to display the temperature in Fahrenheit. (Hint: use `sensors.getTempFByIndex(0)`).
2. **Device Count**: Add code in `setup()` to discover and print the total number of sensors connected on the One-Wire bus. (Hint: use `sensors.getDeviceCount()`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Prints "Error: DS18B20 sensor not found!" | DQ pin wired incorrectly or missing pull-up | Verify that DQ is wired to D2. On physical hardware, verify that the 4.7k pull-up resistor is placed. |
| Output shows `-127` | Sensor disconnected during loop | `-127` corresponds to `DEVICE_DISCONNECTED_C`. Check your wiring connections. |

## Mode Notes
The `OneWire` and `DallasTemperature` libraries are shimmed in the MbedO interpreted mode runtime.

## Related Projects
- [20 - Thermistor Raw Print](../beginner/20-thermistor-raw-print.md)
- [65 - DS18B20 Alarm](65-ds18b20-alarm.md)
- [66 - DS18B20 LCD](66-ds18b20-lcd.md)
