# 171 - Multi-Sensor IoT Node

Read temperature, humidity, gas concentration, and soil moisture simultaneously and stream all sensor values as structured JSON objects over the Serial Monitor — simulating a real IoT edge node data pipeline.

## Goal

Learn how to poll multiple heterogeneous sensors (digital DHT22, analog MQ-2 gas sensor, analog soil moisture sensor) in a single loop, format the collected data as a JSON string, and transmit it over UART at a fixed interval — a foundational pattern used in every real IoT field deployment.

## What You Will Build

A VEGA ARIES v3 board reads ambient temperature and humidity from a DHT22 sensor on GPIO 12, raw gas concentration voltage from an MQ-2 sensor on ADC2 (GP28), and soil moisture voltage from a capacitive soil sensor on ADC3 (GP29). Every 3 seconds, all four values are assembled into a JSON string and printed to the Serial Monitor, simulating an MQTT payload that a cloud gateway would receive.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| MQ-2 Gas Sensor Module | `mq2` | Yes | Yes |
| Capacitive Soil Moisture Sensor | `soil_moisture` | Yes | Yes |
| 10 kΩ Resistor (pull-up for DHT22) | `resistor` | No | Yes |
| Breadboard + jumper wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | 3.3 V power |
| DHT22 Sensor | DATA (Pin 2) | GPIO 12 | Yellow | Digital one-wire data |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground |
| Pull-up Resistor | VCC ↔ DATA | — | — | 10 kΩ between 3V3 and GPIO 12 |
| MQ-2 Module | VCC | 5V | Red | MQ-2 heater requires 5 V |
| MQ-2 Module | GND | GND | Black | Ground |
| MQ-2 Module | AOUT | GP28 (ADC2) | Orange | Analog gas concentration output |
| Soil Moisture Sensor | VCC | 3V3 | Red | 3.3 V power |
| Soil Moisture Sensor | GND | GND | Black | Ground |
| Soil Moisture Sensor | AOUT | GP29 (ADC3) | Green | Analog moisture output |

> **Wiring tip:** The MQ-2 module requires a 5 V supply for its internal heater coil; its analog output signal is typically 0–5 V but the ARIES ADC inputs are 3.3 V max. Use a voltage divider (two 10 kΩ resistors in series, tap between them) on the AOUT line to halve the signal voltage before connecting to GP28. Pre-built MQ-2 breakout boards often include an onboard comparator and a 3.3 V-safe analog output — check your module's datasheet. Allow the MQ-2 at least 60 seconds warm-up time before readings stabilise.

## Code

```cpp
// 171 - Multi-Sensor IoT Node
// Streams DHT22 + MQ-2 + Soil sensor data as JSON over Serial
#include <DHT.h>

#define DHTPIN    12
#define DHTTYPE   DHT22
#define MQ2_PIN   28   // ADC2 = GP28
#define SOIL_PIN  29   // ADC3 = GP29

DHT dht(DHTPIN, DHTTYPE);

// State variables
int   readingId    = 0;
float lastTemp     = 0.0;
float lastHumidity = 0.0;
int   lastGasRaw   = 0;
int   lastSoilRaw  = 0;

void setup() {
  Serial.begin(115200);
  Serial.println("=== Multi-Sensor IoT Node ===");
  Serial.println("Initialising sensors...");
  dht.begin();
  delay(2000); // DHT22 boot settle
  Serial.println("Node ready. Streaming JSON...");
}

void loop() {
  delay(3000);

  // Read DHT22
  float humidity    = dht.readHumidity();
  float temperature = dht.readTemperature();

  // Read analog sensors (0-4095 on 12-bit ADC)
  int gasRaw  = analogRead(MQ2_PIN);
  int soilRaw = analogRead(SOIL_PIN);

  // Validate DHT22 reading
  if (isnan(humidity) || isnan(temperature)) {
    Serial.println("{\"error\":\"DHT22 read failed\"}");
    return;
  }

  // Update state
  lastTemp     = temperature;
  lastHumidity = humidity;
  lastGasRaw   = gasRaw;
  lastSoilRaw  = soilRaw;
  readingId    = readingId + 1;

  // Convert soil raw to percentage (4095 = dry, 0 = wet -- invert for intuitive %)
  int soilPercent = 100 - ((soilRaw * 100) / 4095);

  // Convert gas raw to voltage (3.3 V reference)
  float gasVoltage = (gasRaw * 3.3) / 4095.0;

  // Emit JSON payload
  Serial.print("{");
  Serial.print("\"id\":");
  Serial.print(readingId);
  Serial.print(",\"temp_c\":");
  Serial.print(temperature, 1);
  Serial.print(",\"humidity_pct\":");
  Serial.print(humidity, 1);
  Serial.print(",\"gas_raw\":");
  Serial.print(gasRaw);
  Serial.print(",\"gas_v\":");
  Serial.print(gasVoltage, 3);
  Serial.print(",\"soil_raw\":");
  Serial.print(soilRaw);
  Serial.print(",\"soil_pct\":");
  Serial.print(soilPercent);
  Serial.println("}");
}
```

## What to Click in MbedO

1. Drag the **VEGA ARIES v3**, **DHT22**, **MQ-2**, and **Soil Moisture** components onto the canvas.
2. Wire DHT22 DATA → GPIO 12, VCC → 3V3, GND → GND.
3. Wire MQ-2 AOUT → GP28, VCC → 5V, GND → GND.
4. Wire Soil Moisture AOUT → GP29, VCC → 3V3, GND → GND.
5. Paste the code into the editor and select **Interpreted Mode**.
6. Click **Run**. Wait for "Node ready." to appear in the Serial Monitor.
7. Adjust the temperature/humidity sliders on the DHT22 widget.
8. Move the MQ-2 gas slider and soil moisture slider to simulate different field conditions.
9. Observe the JSON payloads updating in the Serial Monitor every 3 seconds.

## Expected Output

Serial Monitor:
```
=== Multi-Sensor IoT Node ===
Initialising sensors...
Node ready. Streaming JSON...
{"id":1,"temp_c":24.5,"humidity_pct":55.0,"gas_raw":812,"gas_v":0.654,"soil_raw":2100,"soil_pct":49}
{"id":2,"temp_c":24.6,"humidity_pct":55.1,"gas_raw":950,"gas_v":0.765,"soil_raw":1800,"soil_pct":56}
{"id":3,"temp_c":25.0,"humidity_pct":60.0,"gas_raw":3200,"gas_v":2.577,"soil_raw":500,"soil_pct":88}
```

## Expected Canvas Behavior

- Serial Monitor prints a new JSON line every 3 seconds.
- Adjusting any sensor slider causes a visible change in the corresponding JSON field on the next cycle.
- The `id` counter increments with each payload, confirming sequential reads.
- Gas voltage spikes visibly when the MQ-2 slider is pushed to the high-concentration end.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `#include <DHT.h>` | Imports the Adafruit DHT library for one-wire communication with the DHT22. |
| `#define MQ2_PIN 28` | Maps the MQ-2 analog output to ADC2 (GP28). `analogRead(28)` returns 0-4095. |
| `#define SOIL_PIN 29` | Maps the soil sensor to ADC3 (GP29). |
| `DHT dht(DHTPIN, DHTTYPE)` | Creates the sensor driver object pinned to GPIO 12, type DHT22. |
| `int readingId = 0` | Global counter incremented every loop — acts as a packet sequence number. |
| `dht.begin()` | Initialises the DHT22 GPIO and communication timing. |
| `delay(2000)` in setup | Allows DHT22 to fully power up before first read. |
| `delay(3000)` in loop | 3-second sampling interval — DHT22 requires at least 2 s between reads. |
| `dht.readHumidity()` | Returns humidity percentage as a float via the one-wire protocol. |
| `dht.readTemperature()` | Returns temperature in Celsius as a float. |
| `analogRead(MQ2_PIN)` | Reads the 12-bit ADC value from the MQ-2 analog output (0-4095). |
| `isnan(humidity)` | Guards against DHT22 communication failures that return NaN. |
| `soilPercent = 100 - (soilRaw * 100 / 4095)` | Inverts the ADC value so 100% means fully saturated soil. |
| `gasVoltage = (gasRaw * 3.3) / 4095.0` | Converts ADC count to real voltage for calibrated display. |
| `Serial.print("{")` ... `Serial.println("}")` | Constructs a JSON object manually without string buffers. |

## Hardware & Safety Concept

**Sensor Fusion & JSON Serialisation:** A real IoT edge node aggregates data from multiple sensor types — digital (DHT22 uses a custom 1-wire timing protocol delivering 40 bits per measurement) and analog (MQ-2 and soil moisture output a variable voltage proportional to their measured quantity). The microcontroller reads each sensor on its own interface, then serialises the combined results into a structured data format. JSON is the de facto standard for IoT payloads because it is human-readable, self-describing, and natively parseable by cloud platforms such as AWS IoT Core, Azure IoT Hub, and Node-RED. By assigning a sequential `id` to each packet, the receiving system can detect dropped or out-of-order messages — a simple form of data integrity checking.

**MQ-2 Warm-Up Consideration:** Metal-oxide gas sensors like the MQ-2 operate by heating a sensing element to ~300 °C using an internal resistor coil. Accurate readings require at least 60 seconds of warm-up time after power-on. During this window the output voltage drifts significantly. In production systems, a warm-up timer state variable is used to suppress data transmission until stability is confirmed.

## Try This! (Challenges)

1. **Alert Flag in JSON**: Add a global `int alertActive = 0` variable. If `gasVoltage` exceeds 2.0 V, set `alertActive = 1`; otherwise keep it 0. Include `"alert":` in the JSON payload so downstream consumers can filter high-gas events without parsing the voltage field.
2. **Adaptive Sampling Rate**: Use a global `int fastMode = 0` variable. If `gasVoltage > 1.5`, set `fastMode = 1` and use `delay(1000)` instead of `delay(3000)`, switching back when the reading drops below the threshold. Implement this with an `if/else` on `fastMode` at the start of `loop()`.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `"error":"DHT22 read failed"` appears | Loose DATA wire or missing pull-up resistor | Verify GPIO 12 connection and ensure 10 kΩ pull-up is installed between 3V3 and DATA. |
| Gas ADC always reads 0 or 4095 | MQ-2 not powered or output voltage exceeds 3.3 V | Confirm 5 V supply to MQ-2 VCC; add voltage divider on AOUT before GP28. |
| Soil percentage stuck at 100% | Sensor not connected or ADC pin wrong | Check GP29 connection; `analogRead(29)` must correspond to ADC3. |
| JSON lines appear garbled | Baud rate mismatch in Serial Monitor | Set Serial Monitor to 115200 baud to match `Serial.begin(115200)`. |
| `id` counter does not increment | DHT22 fail path returns early every cycle | Fix the DHT22 wiring so `isnan()` guard does not trigger. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [172 - RFID Smart Campus Gate](172-rfid-smart-campus-gate.md)
- [175 - Full Weather Station](175-full-weather-station.md)
- [180 - Automated Greenhouse](180-automated-greenhouse.md)
- [176 - Solar Powered Sensor Node](176-solar-powered-node.md)
