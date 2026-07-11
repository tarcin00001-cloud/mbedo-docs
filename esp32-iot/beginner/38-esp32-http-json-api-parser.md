# 38 - ESP32 HTTP JSON API Parser

Configure the ESP32 to establish an HTTP client connection, fetch a JSON data payload from a remote web service, and deserialize it using the ArduinoJson library to extract specific metadata parameters.

## Goal
Learn how to use the `ArduinoJson` library, parse JSON arrays and key-value objects, configure buffer allocations, and handle deserialization errors.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Every 10 seconds, it fetches a JSON payload from `http://httpbin.org/get`. It parses the JSON, extracts the requester's IP address (from the `origin` key) and the destination URL (from the `url` key), and logs them to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// HTTP JSON API Parser (JSON Deserialization)
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Target URL returning a JSON document
const char* apiURL = "http://httpbin.org/get";

void fetchAndParseJSON() {
  HTTPClient http;
  
  Serial.print("[HTTP] Querying JSON endpoint: ");
  Serial.println(apiURL);
  
  // 1. Initialize HTTP Client connection
  if (!http.begin(apiURL)) {
    Serial.println("[HTTP] Connection failed!");
    return;
  }
  
  // 2. Transmit GET request
  int httpCode = http.GET();
  
  if (httpCode == HTTP_CODE_OK) {
    // 3. Read response payload string
    String payload = http.getString();
    
    // 4. Allocate JSON Document buffer
    // DynamicJsonDocument allocates memory on the heap
    // Allocate 1024 bytes (sufficient for httpbin get metadata response)
    DynamicJsonDocument doc(1024);
    
    // 5. Deserialize the JSON document
    DeserializationError error = deserializeJson(doc, payload);
    
    if (!error) {
      Serial.println("[JSON] Parsing successful!");
      
      // 6. Extract key-value parameters from the parsed document
      const char* originIP = doc["origin"];
      const char* targetURL = doc["url"];
      const char* userAgent = doc["headers"]["User-Agent"]; // Nested object access
      
      Serial.println("--- Parsed API Parameters ---");
      Serial.print("Requester IP (origin): "); Serial.println(originIP);
      Serial.print("Target URL (url):      "); Serial.println(targetURL);
      Serial.print("User Agent header:     "); Serial.println(userAgent);
      Serial.println("-----------------------------");
    } else {
      Serial.print("[JSON] Parsing failed! Error: ");
      Serial.println(error.c_str());
    }
  } else {
    Serial.print("[HTTP] GET failed! Error code: ");
    Serial.println(httpCode);
  }
  
  // 7. Release resources
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing HTTP JSON Parser...");
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
    fetchAndParseJSON();
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Query API every 10 seconds
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the parsed parameters print.

## Expected Output
Serial Monitor:
```
Initializing HTTP JSON Parser...
....
WiFi Connected successfully.
[HTTP] Querying JSON endpoint: http://httpbin.org/get
[JSON] Parsing successful!
--- Parsed API Parameters ---
Requester IP (origin): 10.10.0.3
Target URL (url):      http://httpbin.org/get
User Agent header:     ESP32
-----------------------------
[HTTP] Connection closed.
```

## Expected Canvas Behavior
* The Serial Monitor logs the parsed JSON parameters every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `DynamicJsonDocument doc(...)` | Allocates memory on the heap to store the parsed JSON tree. |
| `deserializeJson(doc, payload)` | Deserializes the JSON string into the document memory buffer. |
| `doc["headers"]["User-Agent"]` | Resolves nested objects (e.g. `headers` -> `User-Agent`) to extract values. |
| `http.end()` | Closes sockets and releases memory. |

## Hardware & Safety Concept: JSON Parsing and Buffer Allocations
IoT devices communicate with online APIs using JSON. Because microcontrollers have limited RAM, parsing large JSON documents can crash the system if not managed correctly:
1. **Buffer Size**: Use `StaticJsonDocument` for small documents (allocated on the stack) and `DynamicJsonDocument` for larger documents (allocated on the heap).
2. **Buffer Limits**: Always verify that the buffer size is sufficient to hold the parsed JSON tree without overflow.

## Try This! (Challenges)
1. **OLED Details Dashboard**: Add an OLED screen (Project 60) and display the parsed IP address.
2. **Dynamic JSON Poster**: Combine this with a POST request (Project 32) to send and verify custom JSON payloads.
3. **Buzzer confirmation tone**: Sound an alert beep (Project 15) if the deserialization fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Deserialization returns `NoMemory` error | Buffer size too small | Increase the document buffer size (e.g. from 1024 to 2048 bytes) |
| Deserialization returns `InvalidInput` | Body is not valid JSON | Print the raw HTTP payload to verify it is formatted correctly |
| The ESP32 crashes on loop | Sockets not closed | Ensure `http.end()` is always called, even if the request fails, to release memory resources |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [39 - ESP32 HTTP API Fetch weather data](39-esp32-http-api-fetch-weather-data-openweathermap-api.md) (Next project)
- [33 - ESP32 HTTPS Secure GET Request](33-esp32-https-secure-get-request.md)
