# 32 - ESP32 HTTP POST Request

Configure the ESP32 to establish an HTTP client connection, format sensor telemetry into an application/x-www-form-urlencoded data string, transmit an HTTP POST request to a remote server, and parse the server response.

## Goal
Learn how to use the `HTTPClient` library, format POST request bodies, configure request headers (such as Content-Type), and parse response payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A potentiometer is on GPIO 34. Every 10 seconds, the ESP32 reads the sensor, formats the data into a form-urlencoded string (e.g. `sensor_val=2048&status=OK`), transmits an HTTP POST request to `http://httpbin.org/post`, and logs the response to the Serial Monitor.

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
// HTTP POST Request (Upload form-urlencoded sensor telemetry)
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/post";

void performHTTPPost() {
  HTTPClient http;
  
  // Read sensor value
  int sensorVal = analogRead(POT_PIN);
  float voltage = sensorVal * (3.3 / 4095.0);
  
  // 1. Format payload as form-urlencoded data
  // Format: key1=value1&key2=value2
  String postData = "sensor_raw=" + String(sensorVal) + 
                    "&voltage=" + String(voltage, 2) + 
                    "&status=ACTIVE";
  
  Serial.print("[HTTP] Connecting to API endpoint: ");
  Serial.println(apiURL);
  
  // 2. Initialize HTTP Client connection
  bool success = http.begin(apiURL);
  
  if (!success) {
    Serial.println("[HTTP] Connection initialization failed!");
    return;
  }
  
  // 3. Configure Content-Type request header
  // Informs the server how to parse the payload
  http.addHeader("Content-Type", "application/x-www-form-urlencoded");
  
  Serial.print("[HTTP] Transmitting POST request payload: ");
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
    Serial.print("[HTTP] POST request failed! Error code: ");
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
  
  Serial.println("Initializing HTTP Client POST Test...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    performHTTPPost();
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Send request every 10 seconds
  delay(10000);
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
Initializing HTTP Client POST Test...
....
WiFi Connected successfully.
[HTTP] Connecting to API endpoint: http://httpbin.org/post
[HTTP] Transmitting POST request payload: sensor_raw=2048&voltage=1.65&status=ACTIVE
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "form": {
    "sensor_raw": "2048", 
    "status": "ACTIVE", 
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
* Adjusting the potentiometer updates the sensor values in the POST payload and the server's response logged to the Serial Monitor every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `postData = ...` | Formats the sensor telemetry into a form-urlencoded data string. |
| `http.addHeader(...)` | Sets the Content-Type header to inform the server how to parse the payload. |
| `http.POST(postData)` | Transmits the payload to the server via an HTTP POST request. |

## Hardware & Safety Concept: HTTP GET vs HTTP POST in Telemetry
* **HTTP GET**: Used to request data from a server. Because GET requests append arguments to the URL query string (e.g. `?temp=24.5`), they are visible in logs and subject to length limits, making them unsuitable for transmitting large payloads.
* **HTTP POST**: Transmits data in the request body. This keeps the data separate from the URL, allowing sending larger payloads (such as images or databases).

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display the upload status.
2. **JSON payload format**: Configure the headers to use `application/json` and transmit a JSON string (e.g. `{"sensor_raw": 2048}`).
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 4) when a transmission completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Server rejects POST request (code 400 or 415) | Content-Type header missing | Verify that `http.addHeader("Content-Type", ...)` is called before sending the request |
| Values missing in server logs | URL format error | Ensure the payload key-value pairs are formatted correctly (separated by `&`) |
| The ESP32 crashes on loop | `http.end()` missing | Ensure `http.end()` is always called, even if the request fails, to release memory resources |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [33 - ESP32 HTTPS Secure GET Request](33-esp32-https-secure-get-request.md)
- [49 - ESP32 Wireless DHT22 logger](49-esp32-wireless-dht22-logger.md)
