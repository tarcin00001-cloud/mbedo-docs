# 120 - ESP32 Bluetooth DHT22 Telemetry Broadcast

Build a wireless environmental telemetry node that reads temperature and humidity from a DHT22 sensor and broadcasts the formatted logs to a paired Bluetooth device.

## Goal
Learn how to transmit structured sensor telemetry packets wirelessly over a Bluetooth serial link, and handle incoming command queries.

## What You Will Build
A DHT22 sensor is connected to GPIO 4. An HC-05 Bluetooth module is connected to UART2 (RX2: GPIO 16, TX2: GPIO 17). Every 3 seconds, the ESP32 broadcasts the current temperature and humidity data over the Bluetooth serial channel. It also responds to specific query commands ("GET_T", "GET_H") sent from a mobile app.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Climate telemetry input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| HC-05 Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| HC-05 Module | TXD | GPIO16 (RX2) | Green | Bluetooth Transmit |
| HC-05 Module | RXD | GPIO17 (TX2) | Yellow | Bluetooth Receive |

> **Wiring tip:** Standard HC-05 modules operate on 9600 baud in communication mode. Ensure the RXD and TXD lines are connected to GPIO 16 and 17.

## Code
```cpp
// Bluetooth DHT22 Telemetry Broadcast
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22

#define RX2_PIN 16
#define TX2_PIN 17

DHT dht(DHTPIN, DHTTYPE);

unsigned long lastBroadcastTime = 0;
const unsigned long BROADCAST_INTERVAL_MS = 3000; // Broadcast every 3s

String inputCmd = "";

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  dht.begin();
  
  Serial.println("Bluetooth DHT22 Telemetry active.");
}

void loop() {
  unsigned long now = millis();
  
  // 1. Periodic Telemetry Broadcast
  if (now - lastBroadcastTime >= BROADCAST_INTERVAL_MS) {
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();
    
    if (!isnan(temp) && !isnan(hum)) {
      // Format telemetry packet
      String packet = "TELEMETRY: T=" + String(temp, 1) + "C, H=" + String(hum, 1) + "%";
      
      // Broadcast over Bluetooth
      Serial2.println(packet);
      
      // Mirror to USB Serial Monitor
      Serial.print("Broadcast: "); Serial.println(packet);
    } else {
      Serial2.println("ERROR: Sensor read failed");
    }
    
    lastBroadcastTime = now;
  }
  
  // 2. Process incoming command queries over Bluetooth
  while (Serial2.available() > 0) {
    char inChar = (char)Serial2.read();
    
    if (inChar == '\n' || inChar == '\r') {
      inputCmd.trim();
      if (inputCmd.length() > 0) {
        handleBluetoothQuery(inputCmd);
      }
      inputCmd = ""; // Clear buffer
    } else {
      inputCmd += inChar;
    }
  }
}

void handleBluetoothQuery(String query) {
  Serial.print("Received Query: "); Serial.println(query);
  
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  
  if (isnan(temp) || isnan(hum)) {
    Serial2.println("ERR: Sensor offline");
    return;
  }
  
  if (query.equalsIgnoreCase("GET_T")) {
    Serial2.print("TEMP: "); Serial2.print(temp, 1); Serial2.println(" C");
  } 
  else if (query.equalsIgnoreCase("GET_H")) {
    Serial2.print("HUMID: "); Serial2.print(hum, 1); Serial2.println(" %");
  } 
  else {
    Serial2.println("ERR: Unknown Query. Use GET_T or GET_H");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, and **HC-05 Module** onto the canvas.
2. Wire DHT22 DATA to **GPIO4**, and HC-05 TXD/RXD to **GPIO16/GPIO17**.
3. Paste the code and click **Run**.
4. Observe periodic telemetry packets printed in the virtual Bluetooth console.
5. Type `GET_T` in the Bluetooth input field to request current temperature.

## Expected Output
Serial Monitor:
```
Bluetooth DHT22 Telemetry active.
Broadcast: TELEMETRY: T=24.5C, H=45.2%
Received Query: GET_T
Broadcast: TELEMETRY: T=24.5C, H=45.2%
```

Bluetooth Terminal:
```
TELEMETRY: T=24.5C, H=45.2%
TEMP: 24.5 C
TELEMETRY: T=24.5C, H=45.2%
```

## Expected Canvas Behavior
* Telemetry packets broadcast automatically every 3 seconds to the paired Bluetooth console widget.
* Typing `GET_T\n` or `GET_H\n` in the Bluetooth widget immediately returns the corresponding individual climate values.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `String(temp, 1)` | Converts a floating-point temperature value into a string rounded to 1 decimal place. |
| `Serial2.println(packet)` | Transmits the telemetry data string over the Bluetooth serial channel. |
| `query.equalsIgnoreCase("GET_T")` | Parses query commands requested by remote client apps. |

## Hardware & Safety Concept: Telemetry Scheduling and Airtime Congestion
Telemetry systems transmit packets on fixed intervals to reduce bandwidth congestion and battery usage. Continuous streaming can flood serial buffers, causing latency spikes in receiver devices. Standard environmental nodes use broadcast rates of 0.1Hz to 1Hz (once every 1 to 10 seconds), conserving energy and channel airtime.

## Try This! (Challenges)
1. **Dynamic Alert Thresholds**: Send warning strings over Bluetooth ("WARNING: HOT!") if the temperature climbs above 30 °C.
2. **Status LED Indicator**: Flash an LED on GPIO 5 every time a telemetry packet is successfully transmitted.
3. **Data Request Ack**: Require client devices to send a handshake code before broadcasting to authenticate connections.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Telemetry shows Nan values | DHT22 sensor offline or pin mapping error | Check DHT22 DATA connection to GPIO 4 |
| Telemetry packets print garbage letters | Baud rate mismatch | Ensure Bluetooth serial baud rate is matching `9600` |
| Responses lag behind queries | Long blocking delays in loops | Avoid adding long `delay()` functions inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](71-esp32-dht22-temp-humidity-serial-logs.md)
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](89-esp32-hc05-bluetooth-serial-uart-link.md)
- [90 - ESP32 Bluetooth LED Controller](90-esp32-hc05-bluetooth-led-controller-string-parsing.md)
