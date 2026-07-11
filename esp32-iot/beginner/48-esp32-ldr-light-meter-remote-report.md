# 48 - ESP32 LDR Light Meter Remote Report

Configure the ESP32 to connect to a local WiFi network, measure ambient light levels using an analog photoresistor (LDR) on GPIO 34, and transmit the readings via an HTTP POST request to a remote server.

## Goal
Learn how to sample LDR photoresistors, map raw ADC inputs to percentages, determine day/night states, and transmit data.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LDR (photoresistor) is on GPIO 34. Every 5 seconds, the ESP32 reads the sensor, maps the value to a percentage (0% to 100%), determines the day/night status, formats the data into a form-urlencoded string (e.g. `ldr_raw=2048&light_percent=50&status=DAYLIGHT`), transmits an HTTP POST request to `http://httpbin.org/post`, and logs the response to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Photoresistor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Light sensor analog input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | Black | Divider pull-down resistor |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** The photoresistor requires a 10 kΩ pull-down resistor to ground to form a voltage divider. This maps dark to low voltage (low raw ADC values) and light to high voltage (high raw ADC values).

## Code
```cpp
// LDR light meter remote report (Upload ambient light levels via HTTP POST)
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/post";

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 5000; // Upload every 5 seconds

void uploadTelemetry() {
  HTTPClient http;
  
  // Read photoresistor value (0 to 4095)
  int ldrRaw = analogRead(LDR_PIN);
  
  // Map raw ADC to light percentage (0% to 100%)
  int lightPercent = map(ldrRaw, 0, 4095, 0, 100);
  
  // Determine Day/Night status
  String statusStr = (lightPercent > 50) ? "DAYLIGHT" : "NIGHT";
  
  // 1. Format payload as form-urlencoded data
  String postData = "ldr_raw=" + String(ldrRaw) + 
                    "&light_percent=" + String(lightPercent) + 
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
  
  pinMode(LDR_PIN, INPUT);
  
  Serial.println("Initializing Wireless LDR Uploader...");
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
1. Drag **ESP32 DevKitC** and **LDR** onto the canvas.
2. Wire LDR to **GPIO34**.
3. Paste the code and click **Run**.
4. Adjust the LDR light level slider. Watch the printed POST payload and the server's response.

## Expected Output
Serial Monitor:
```
Initializing Wireless LDR Uploader...
....
WiFi Connected successfully.
[HTTP] Connecting to: http://httpbin.org/post
[HTTP] Transmitting telemetry payload: ldr_raw=3200&light_percent=78&status=DAYLIGHT
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "form": {
    "ldr_raw": "3200", 
    "light_percent": "78", 
    "status": "DAYLIGHT"
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
* Adjusting the LDR slider updates the light levels in the POST payload and the server's response logged to the Serial Monitor every 5 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(LDR_PIN)` | Samples the analog voltage on the photoresistor wiper pin. |
| `map(...)` | Maps the raw ADC reading to a percentage (0% to 100%). |
| `http.POST(postData)` | Transmits the payload to the server via an HTTP POST request. |
| `http.end()` | Closes sockets and releases memory. |

## Hardware & Safety Concept: Sensor Calibration and Subnets
LDR photoresistors read different values based on the resistor used in the divider. Calibration maps raw values (e.g. 0 to 4095) to percentage ranges (0% to 100%) to establish a standardized scale. This ensures the day/night threshold remains accurate.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current light level percentage.
2. **JSON payload format**: Configure the headers to use `application/json` and transmit a JSON string (e.g. `{"light_percent": 78}`).
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
- [49 - ESP32 Wireless DHT22 logger](49-esp32-wireless-dht22-logger.md)
