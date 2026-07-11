# 31 - ESP32 HTTP GET Request

Configure the ESP32 to establish an HTTP client connection, transmit a Hypertext Transfer Protocol (HTTP) GET request to a remote web server, and print the returned status codes and response body to the Serial Monitor.

## Goal
Learn how to use the `HTTPClient` library, format GET requests, parse HTTP status codes (such as 200 OK or 404 Not Found), and extract string payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Every 10 seconds, it initiates an HTTP GET request to `http://httpbin.org/get`, logs the response status code, and prints the JSON body containing request metadata to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// HTTP GET Request (Fetch web telemetry)
#include <WiFi.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Target URL API endpoint
const char* apiURL = "http://httpbin.org/get";

void performHTTPGet() {
  HTTPClient http;
  
  Serial.print("[HTTP] Connecting to API endpoint: ");
  Serial.println(apiURL);
  
  // 1. Initialize HTTP Client connection
  // Takes the destination URL string
  bool success = http.begin(apiURL);
  
  if (!success) {
    Serial.println("[HTTP] Connection initialization failed!");
    return;
  }
  
  Serial.println("[HTTP] Transmitting GET request...");
  
  // 2. Transmit GET request
  // Returns the HTTP status code (positive) or error code (negative)
  int httpResponseCode = http.GET();
  
  // 3. Evaluate Response
  if (httpResponseCode > 0) {
    Serial.print("[HTTP] Response Status Code: ");
    Serial.println(httpResponseCode); // e.g. 200 for OK
    
    // 4. Retrieve response payload
    String payload = http.getString();
    Serial.println("[HTTP] --- Response Body ---");
    Serial.println(payload);
    Serial.println("[HTTP] ---------------------");
  } else {
    Serial.print("[HTTP] GET request failed! Error code: ");
    Serial.println(http.errorToString(httpResponseCode).c_str());
  }
  
  // 5. Release resources
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing HTTP Client GET Test...");
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
    performHTTPGet();
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Send request every 10 seconds
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the JSON response prints.

## Expected Output
Serial Monitor:
```
Initializing HTTP Client GET Test...
....
WiFi Connected successfully.
[HTTP] Connecting to API endpoint: http://httpbin.org/get
[HTTP] Transmitting GET request...
[HTTP] Response Status Code: 200
[HTTP] --- Response Body ---
{
  "args": {}, 
  "headers": {
    "Host": "httpbin.org", 
    "User-Agent": "ESP32"
  }, 
  "url": "http://httpbin.org/get"
}
[HTTP] ---------------------
[HTTP] Connection closed.
```

## Expected Canvas Behavior
* The Serial Monitor logs the HTTP status codes and the JSON body every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `HTTPClient http` | Instantiates an HTTP client utility helper object. |
| `http.begin(apiURL)` | Configures the destination address and initializes transport protocols. |
| `http.GET()` | Transmits the HTTP GET request, blocking until headers are received. |
| `http.getString()` | Reads the response body stream into a local string object. |
| `http.end()` | Closes TCP sockets and releases memory. |

## Hardware & Safety Concept: HTTP Client Resource Management
Unlike raw TCP socket clients, the `HTTPClient` library abstracts header generation and chunked parsing. When using HTTP clients on microcontrollers, always call `http.end()` after completing a request. This ensures that socket handles are closed, preventing memory exhaustion (heap fragmentation) that can crash the ESP32 after a few hours of operation.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the HTTP status code.
2. **Dynamic URL Query**: Append custom arguments to the URL query string (e.g. `http://httpbin.org/get?temp=24.5`).
3. **Buzzer Error chime**: Sound an alarm beep (Project 15) if the response code is anything other than 200.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails immediately | DNS lookup failed | Verify that the target server hostname is typed correctly |
| Response returns negative codes | TCP connection timeout | Check your router's gateway routing and internet access |
| Memory leaks / crashes | `http.end()` missing | Ensure `http.end()` is always called, even if the request fails |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 TCP Client connection test](16-esp32-tcp-client-connection-test.md)
- [32 - ESP32 HTTP POST Request](32-esp32-http-post-request.md)
- [33 - ESP32 HTTPS Secure GET Request](33-esp32-https-secure-get-request.md)
