# 194 - Crop Health Analyzer

Build a micro-agricultural analysis node that measures soil moisture, ambient light, temperature, humidity, and simulated NPK (Nitrogen, Phosphorus, Potassium) levels, formatting the aggregated soil health metrics into a serialized JSON string for transmission over the UART Serial interface.

## Goal
Learn how to sample multiple analog and digital sensors, map raw data to agricultural percentage scales, package complex nested telemetry structures into JSON formats using print calls without helper functions, and avoid arrays or loops in interpreted execution.

## What You Will Build
A farm telemetry station. The ARIES v3 board reads temperature and humidity from a DHT22, light from an LDR (ADC1), soil moisture from a resistive soil probe (ADC3), and nitrogen/phosphorus levels from analog inputs ADC0 and ADC2. Potassium is mathematically modeled. Every 3 seconds, the board formats these readings into a single-line JSON object and streams it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `soil_moisture` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| 3x Analog Input sources (NPK simulator) | `analog_sensor` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Power input |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| DHT22 Sensor | DATA | GPIO 12 | Yellow | One-wire data bus |
| Soil Moisture | VCC | 3V3 | Red | Probe excitation power |
| Soil Moisture | GND | GND | Black | Ground reference |
| Soil Moisture | Analog Out | ADC3 (GP29) | Blue | Soil moisture voltage level |
| LDR | Pin 1 | 3V3 | Red | Light sensor power |
| LDR | Pin 2 | ADC1 (GP27) | White | Light intensity output (with 10k to GND) |
| Nitrogen Probe | AO | ADC0 (GP26) | Orange | Nitrogen sensor input |
| Phosphorus Probe| AO | ADC2 (GP28) | Green | Phosphorus sensor input |

> **Wiring tip:** Soil moisture probes corrode quickly (electrolysis) if left continuously powered. In real-world designs, the probe's VCC is connected to a digital GPIO pin instead of the 3.3V rail. The GPIO pin is pulled HIGH only during a read and then pulled LOW to preserve probe life.

## Code
```cpp
// 194 - Crop Health Analyzer
#include <Wire.h>
#include <DHT.h>

#define DHTPIN 12
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);

const int SOIL_PIN = 29;  // ADC3
const int LDR_PIN = 27;   // ADC1
const int NITROGEN_PIN = 26; // ADC0
const int PHOS_PIN = 28;  // ADC2

float temp = 0.0;
float hum = 0.0;
int rawSoil = 0;
float soilPercent = 0.0;
int lightVal = 0;

int rawNitrogen = 0;
int rawPhos = 0;
float nitrogenPpm = 0.0;
float phosPpm = 0.0;
float potassiumPpm = 0.0;

int logTimer = 0;

void setup() {
  Serial.begin(115200);
  dht.begin();
  
  // Configure pin modes for onboard LEDs to indicate status
  pinMode(23, OUTPUT); // Red
  pinMode(24, OUTPUT); // Green
  pinMode(25, OUTPUT); // Blue

  // Initialize LEDs OFF (Active LOW)
  digitalWrite(23, HIGH);
  digitalWrite(24, HIGH);
  digitalWrite(25, HIGH);

  Serial.println("Crop Health Analyzer Online.");
}

void loop() {
  // Read DHT22
  float rTemp = dht.readTemperature();
  float rHum = dht.readHumidity();
  if (!isnan(rTemp)) temp = rTemp;
  if (!isnan(rHum)) hum = rHum;

  // Read Soil Moisture and scale to 0-100% (dry to wet)
  rawSoil = analogRead(SOIL_PIN);
  // ARIES ADC: 1023 is dry, 0 is wet in some probes; map accordingly.
  // Assuming standard scaling: higher reading = wetter soil
  soilPercent = (rawSoil / 1023.0) * 100.0;

  // Read Light Level
  lightVal = analogRead(LDR_PIN);

  // Read NPK simulated levels (0-3.3V mapped to ppm values, e.g. 0-200 ppm)
  rawNitrogen = analogRead(NITROGEN_PIN);
  rawPhos = analogRead(PHOS_PIN);
  
  nitrogenPpm = (rawNitrogen / 1023.0) * 200.0;
  phosPpm = (rawPhos / 1023.0) * 200.0;
  
  // Simulate Potassium as a function of Nitrogen and Phosphorus for multi-sensor display
  potassiumPpm = (nitrogenPpm + phosPpm) * 0.75;
  if (potassiumPpm > 200.0) {
    potassiumPpm = 200.0;
  }

  // Update status RGB LED based on soil moisture (Blue = Wet, Green = Normal, Red = Dry)
  if (soilPercent < 30.0) {
    digitalWrite(23, LOW);  // Red LED ON
    digitalWrite(24, HIGH);
    digitalWrite(25, HIGH);
  } else if (soilPercent >= 30.0 && soilPercent < 70.0) {
    digitalWrite(23, HIGH);
    digitalWrite(24, LOW);  // Green LED ON
    digitalWrite(25, HIGH);
  } else {
    digitalWrite(23, HIGH);
    digitalWrite(24, HIGH);
    digitalWrite(25, LOW);  // Blue LED ON
  }

  // Generate serialized JSON output every 3 seconds (15 * 200ms)
  logTimer++;
  if (logTimer >= 15) {
    logTimer = 0;

    // Package metrics as a JSON string using individual serial writes (avoiding sprint buffer arrays)
    Serial.print("{\"temp\":");
    Serial.print(temp, 1);
    Serial.print(",\"hum\":");
    Serial.print(hum, 1);
    Serial.print(",\"soil\":");
    Serial.print(soilPercent, 1);
    Serial.print(",\"light\":");
    Serial.print(lightVal);
    Serial.print(",\"N\":");
    Serial.print(nitrogenPpm, 1);
    Serial.print(",\"P\":");
    Serial.print(phosPpm, 1);
    Serial.print(",\"K\":");
    Serial.print(potassiumPpm, 1);
    Serial.println("}");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, **Soil Moisture Sensor**, **LDR Photoresistor**, and two **analog sources** (for Nitrogen/Phosphorus simulation) onto the canvas.
2. Wire the DHT22: **VCC → 3V3**, **GND → GND**, **DATA → GPIO 12**.
3. Wire the Soil Moisture sensor: **Analog Out → ADC3 (GP29)**.
4. Wire the LDR: **Output → ADC1 (GP27)** (with 10k to GND).
5. Wire the Nitrogen simulator source: **Analog Out → ADC0 (GP26)**; Phosphorus simulator: **Analog Out → ADC2 (GP28)**.
6. Paste the code, select **Interpreted Mode**, and click **Run**.
7. Adjust the sensor sliders. Watch the JSON strings update in the Serial Monitor.

## Expected Output
Serial Monitor:
```
Crop Health Analyzer Online.
{"temp":24.5,"hum":45.0,"soil":62.5,"light":680,"N":42.5,"P":31.0,"K":55.1}
{"temp":24.6,"hum":45.2,"soil":61.8,"light":682,"N":42.5,"P":31.0,"K":55.1}
{"temp":24.5,"hum":45.0,"soil":28.4,"light":675,"N":12.0,"P":8.5,"K":15.4}
```

## Expected Canvas Behavior
* Startup: Onboard RGB LED initializes to green (if soil moisture is within the 30% - 70% range).
* Decreasing soil moisture below 30% turns the RGB LED Red.
* Every 3 seconds, a new JSON payload containing the structured data outputs on a single line in the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Polls the digital temperature sensor using the 1-wire library. |
| `analogRead(SOIL_PIN)` | Samples the resistive soil sensor voltage on GP29. |
| `soilPercent = ...` | Scales raw sensor readings to a zero-to-hundred percent range. |
| `nitrogenPpm = ...` | Converts raw Nitrogen ADC input to parts-per-million (0-200 ppm). |
| `Serial.print("{\"temp\":")` | Begins manual construction of the JSON character stream. |
| `Serial.print(temp, 1)` | Writes decimal temperature float with single digit precision. |

## Hardware & Safety Concept
* **JSON Serial Communication:** JSON (JavaScript Object Notation) is the standard data serialization format for IoT web APIs. Packing data into structured JSON objects allows receiving gateways (Raspberry Pi, Cloud database) to easily parse values using generic key names rather than hardcoded index splits.
* **NPK Agricultural Chemistry:** Macronutrients Nitrogen (N), Phosphorus (P), and Potassium (K) dictate crop health. In farming, optical spectral sensors or electrochemical probes measure these levels. Because they are delicate, hardware buffers are used to filter ground loop currents from interfering with ADC readings.

## Try This! (Challenges)
1. **Soil Moisture Threshold Alerts:** Connect a buzzer to GPIO 14. Beep in a slow cadence if the soil moisture drops below 25% to alert the farmer to water the crop.
2. **Dynamic Log Rates:** Change the logging frequency based on activity: if light levels drop below 100 (night time), change the JSON printing rate to once every 10 seconds to conserve wireless bandwidth.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| JSON output contains NaN values | DHT22 sensor failed to read | Verify DATA pin is connected to GPIO 12 and verify slider configuration. |
| Soil moisture reads constant 100% | Probe short circuit or incorrect wiring | Check GP29 and ensure the sensor has a shared ground reference. |
| JSON formatting is broken | Misplaced quotation marks in code | Match the exact print statements. Escape double quotes `\"` correctly inside print blocks. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - DHT22 Temp & Humidity Serial Logs](../intermediate/71-dht22-temp-humidity-serial.md)
- [110 - Soil Moisture Automated Irrigation](../intermediate/110-soil-moisture-automated-irrigation.md)
