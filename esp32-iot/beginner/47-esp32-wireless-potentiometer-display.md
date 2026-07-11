# 47 - ESP32 Wireless Potentiometer Display (ADC -> Remote Server)

Configure the ESP32 to connect to a local WiFi network, read an analog potentiometer sensor using the internal Analog-to-Digital Converter (ADC) on GPIO 34, and transmit the raw readings via an HTTP POST request to a remote server.

## Goal
Learn how to sample analog inputs, format data payloads, configure request headers, and transmit telemetry data via HTTP POST requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A potentiometer is on GPIO 34. Every 5 seconds, the ESP32 reads the sensor, formats the data into a form-urlencoded string (e.g. `sensor_raw=2048&uptime=10`), transmits an HTTP POST request to `http://httpbin.org/post`, and logs the response to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor input |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// Wireless Potentiometer display (Upload potentiometer ADC levels via HTTP POST)
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/post";

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 5000; // Upload every 5 seconds

void uploadTelemetry() {
  HTTPClient http;
  
  // Read potentiometer value (0 to 4095)
  int potRaw = analogRead(POT_PIN);
  float voltage = potRaw * (3.3 / 4095.0);
  
  // 1. Format payload as form-urlencoded data
  String postData = "sensor_raw=" + String(potRaw) + 
                    "&voltage=" + String(voltage, 2) + 
                    "&uptime_s=" + String(millis() / 1000);
  
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
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("Initializing Wireless Potentiometer Uploader...");
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
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Watch the printed POST payload and the server's response.

## Expected Output
Serial Monitor:
```
Initializing Wireless Potentiometer Uploader...
....
WiFi Connected successfully.
[HTTP] Connecting to: http://httpbin.org/post
[HTTP] Transmitting telemetry payload: sensor_raw=2048&voltage=1.65&uptime_s=5
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "form": {
    "sensor_raw": "2048", 
    "uptime_s": "5", 
    "voltage": "1.65"
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
* Adjusting the potentiometer updates the sensor values in the POST payload and the server's response logged to the Serial Monitor every 5 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(POT_PIN)` | Samples the analog voltage on the potentiometer wiper pin. |
| `http.addHeader(...)` | Sets the Content-Type header to inform the server how to parse the payload. |
| `http.POST(postData)` | Transmits the payload to the server via an HTTP POST request. |
| `http.end()` | Closes sockets and releases memory. |

## Hardware & Safety Concept: WiFi Power Consumption during Transmissions
The ESP32 WiFi radio consumes significant current (up to 240 mA) during transmissions. If a device queries the server continuously, it will exhaust battery power quickly. Implementing a **transmission interval** (e.g. once every 5 seconds or 1 minute) allows the radio to sleep between transmissions, reducing power consumption.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current sensor value.
2. **JSON payload format**: Configure the headers to use `application/json` and transmit a JSON string (e.g. `{"sensor_raw": 2048}`).
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
- [48 - ESP32 LDR light meter remote report](48-esp32-ldr-light-meter-remote-report.md) (Next project)
- [49 - ESP32 Wireless DHT22 logger](49-esp32-wireless-dht22-logger.md)
