# 71 - ESP32 DHT22 Temp & Humidity Serial Logs

Read ambient temperature and relative humidity from a DHT22 sensor and print the structured data logs to the Serial Monitor.

## Goal
Learn how to interface the digital DHT22 sensor using the `DHT` library, execute a one-wire protocol read, and parse temperature (°C) and humidity (%) values.

## What You Will Build
A DHT22 sensor is connected to GPIO 4. The ESP32 queries the sensor every 2 seconds, extracts temperature and humidity readings, and prints formatted logs to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| 10 kΩ Resistor (pull-up) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | Power input |
| DHT22 Sensor | DATA (Pin 2) | GPIO4 | Yellow | Bidirectional data line |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground reference |
| Pull-up Resistor | VCC to DATA | — | — | Place 10k resistor between VCC and DATA pins |

> **Wiring tip:** Most DHT22 sensor modules (mounted on a small PCB) already contain the necessary 10 kΩ pull-up resistor on the data line. If you are using a bare 4-pin DHT22 sensor, you must place an external 10 kΩ resistor between Pin 1 (VCC) and Pin 2 (DATA). Pin 3 of the DHT22 is not connected.

## Code
```cpp
// DHT22 Temperature & Humidity Serial Logs
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);

void setup() {
  Serial.begin(115200);
  Serial.println("DHT22 Sensor Initialization...");
  
  dht.begin();
  Serial.println("DHT22 Sensor ready.");
}

void loop() {
  // Wait 2 seconds between measurements (DHT22 reading rate is ~0.5Hz)
  delay(2000);
  
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature(); // Celsius
  
  // Check if any reads failed
  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("Failed to read from DHT22 sensor!");
    return;
  }
  
  // Compute Heat Index in Celsius
  float heatIndex = dht.computeHeatIndex(temperature, humidity, false);

  Serial.print("Humidity: ");
  Serial.print(humidity, 1);
  Serial.print("%  |  ");
  Serial.print("Temperature: ");
  Serial.print(temperature, 1);
  Serial.print(" °C  |  ");
  Serial.print("Heat Index: ");
  Serial.print(heatIndex, 1);
  Serial.println(" °C");
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22** onto the canvas.
2. Connect DHT22 **DATA** to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the temperature and humidity sliders on the DHT22 widget and observe the updated values in the Serial Monitor.

## Expected Output
Serial Monitor:
```
DHT22 Sensor Initialization...
DHT22 Sensor ready.
Humidity: 45.2%  |  Temperature: 24.5 °C  |  Heat Index: 24.1 °C
Humidity: 60.0%  |  Temperature: 28.0 °C  |  Heat Index: 29.5 °C
```

## Expected Canvas Behavior
* The Serial Monitor prints updated sensor logs every 2 seconds.
* Moving the sliders on the DHT22 component updates the output instantly on the next loop cycle.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `#include <DHT.h>` | Imports the Adafruit DHT Sensor Library. |
| `dht.begin()` | Sets up the GPIO pin configurations for communicating with the DHT22. |
| `dht.readHumidity()` | Reads the digital humidity percentage (returns a float). |
| `dht.readTemperature()` | Reads the digital temperature in Celsius (returns a float). |
| `isnan(...)` | Verifies that the returned data is a valid number and not NaN (Not a Number). |

## Hardware & Safety Concept: One-Wire Digital Sensor Protocol
Unlike analog sensors that output a continuous voltage level, the DHT22 uses a custom single-wire digital protocol. The ESP32 triggers a start signal by pulling the data line LOW for 18 ms, then releasing it. The DHT22 responds with a handshake followed by 40 bits of data (16 bits humidity, 16 bits temperature, 8 bits checksum). The library automatically decodes these pulses. Digital transmission prevents signal degradation over long wire lengths compared to analog sensor signals.

## Try This! (Challenges)
1. **Fahrenheit Logging**: Modify the code to print the temperature in Fahrenheit by passing `true` to `dht.readTemperature(true)`.
2. **Min/Max Tracker**: Add logic to track and print the maximum and minimum temperatures recorded since startup.
3. **Sensor Reading Average**: Average 5 consecutive readings to smooth out short-term fluctuations.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "Failed to read from DHT22 sensor!" | Missing pull-up resistor or incorrect GPIO pin | Verify DATA is connected to GPIO 4; check if using a module with built-in pull-up |
| Reads are always constant | Querying the sensor too fast | Make sure `delay(2000)` is included in the loop; DHT22 requires a 2-second recovery window |
| Readings show NaN | Loose connection | Re-seat wires or swap DHT22 module |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [72 - ESP32 DHT22 Temperature Alarm](72-esp32-dht22-temperature-alarm.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
- [74 - ESP32 DHT22 Temperature OLED Graph](74-esp32-dht22-temperature-oled-graph.md)
