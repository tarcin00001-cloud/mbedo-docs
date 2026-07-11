# 49 - ESP32 Wireless DHT22 Logger

Configure the ESP32 to connect to a local WiFi network, read temperature and humidity levels from a DHT22 sensor on GPIO 4, and transmit the readings via an HTTP POST request to a remote server.

## Goal
Learn how to sample DHT22 sensors, format float telemetry values, determine overheat alarm states, and transmit data.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 4. Every 10 seconds, the ESP32 reads the temperature and humidity, determines the status (normal or warning), formats the data into a form-urlencoded string (e.g. `temperature=24.5&humidity=45.0&status=NORMAL`), transmits an HTTP POST request to `http://httpbin.org/post`, and logs the response to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp & Humidity input |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the DHT22 data pin directly to GPIO 4.

## Code
```cpp
// Wireless DHT22 logger (Upload climate levels via HTTP POST)
#include <WiFi.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 4;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/post";

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 10000; // Upload every 10 seconds

void uploadTelemetry() {
  HTTPClient http;
  
  // Read temperature and humidity
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  
  if (isnan(temp) || isnan(hum)) {
    Serial.println("Error: Failed to read from DHT sensor!");
    return;
  }
  
  // Determine Overheat Warning status
  String statusStr = (temp > 35.0) ? "WARNING_OVERHEAT" : "NORMAL";
  
  // 1. Format payload as form-urlencoded data
  String postData = "temperature=" + String(temp, 1) + 
                    "&humidity=" + String(hum, 1) + 
                    "&status=" + statusStr;
  
  Serial.print("[HTTP] Connecting to: ");
  Serial.println(apiURL);
  
  // 2. Initialize HTTP Client connection
  if (!http.begin(apiURL)) {
    Serial.println("[HTTP] Connection initialization failed!");
    return;
  }
  
  // 3. Configure Content-Type request header
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");
  
  Serial.print("[HTTP] Transmitting telemetry payload: ");
  Serial.println(postData);
  
  // 4. Transmit POST request
  int httpResponseCode = http.POST(postData);
  
  // 5. Evaluate Response
  if (httpResponseCode > 0) {
    Serial.print("[HTTP] Response Status Code: ");
    Serial.println(httpResponseCode);
    
    // Retrieve response body
    String payload = http.getString();
    Serial.println("[HTTP] --- Response Body ---");
    Serial.println(payload);
    Serial.println("[HTTP] ---------------------");
  } else {
    Serial.print("[HTTP] POST failed! Error: ");
    Serial.println(http.errorToString(httpResponseCode).c_str());
  }
  
  // 6. Release resources
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  Serial.println("Initializing Wireless DHT Uploader...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Initial upload
  uploadTelemetry();
  lastUploadTime = millis();
}

void loop() {
  unsigned long now = millis();
  
  // Periodically upload data
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    if (WiFi.status() == WL_CONNECTED) {
      uploadTelemetry();
    } else {
      Serial.println("WiFi offline!");
    }
    lastUploadTime = now;
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **DHT22** onto the canvas.
2. Wire DHT22 to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the DHT22 temperature and humidity sliders. Watch the printed POST payload and the server's response.

## Expected Output
Serial Monitor:
```
Initializing Wireless DHT Uploader...
....
WiFi Connected successfully.
[HTTP] Connecting to: http://httpbin.org/post
[HTTP] Transmitting telemetry payload: temperature=24.5&humidity=45.0&status=NORMAL
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "form": {
    "humidity": "45.0", 
    "status": "NORMAL", 
    "temperature": "24.5"
  }, 
  "headers": {
    "Content-Type": "application/x-www-form-urlencoded", 
    "Host": "httpbin.org"
  }, 
  "url": "http://httpbin.org/post"
}
[HTTP] ---------------------
[HTTP] Connection closed.
```

## Expected Canvas Behavior
* Adjusting the DHT22 sliders updates the temperature and humidity values in the POST payload and the server's response logged to the Serial Monitor every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Reads the temperature value in degrees Celsius from the DHT22 sensor. |
| `http.addHeader(...)` | Sets the Content-Type header to inform the server how to parse the payload. |
| `http.POST(postData)` | Transmits the payload to the server via an HTTP POST request. |
| `http.end()` | Closes sockets and releases memory. |

## Hardware & Safety Concept: Telemetry Upload Frequency
Bridges and routers can drop packets if a device floods the network with high-frequency uploads. Telemetry values like temperature change slowly over time. Setting an **upload interval** (e.g. once every 10 seconds or 1 minute) reduces network congestion and prevents rate limiting from servers.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current temperature and humidity.
2. **JSON payload format**: Configure the headers to use `application/json` and transmit a JSON string (e.g. `{"temperature": 24.5}`).
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 4) when a transmission completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Server rejects POST request (code 400 or 415) | Content-Type header missing | Verify that `http.addHeader("Content-Type", ...)` is called before sending the request |
| Values missing in server logs | URL format error | Ensure the payload key-value pairs are formatted correctly (separated by `&`) |
| The ESP32 crashes on loop | Sockets not closed | Ensure `http.end()` is always called, even if the request fails, to release memory resources |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [32 - ESP32 HTTP POST Request](32-esp32-http-post-request.md)
- [47 - ESP32 Wireless Potentiometer display](47-esp32-wireless-potentiometer-display.md)
- [48 - ESP32 LDR light meter remote report](48-esp32-ldr-light-meter-remote-report.md)
