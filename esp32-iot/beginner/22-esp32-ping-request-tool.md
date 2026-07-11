# 22 - ESP32 Ping Request Tool

Build a network latency testing utility (TCP Ping) on the ESP32 that measures the round-trip handshake time to a target host on port 80 and logs the response time in milliseconds to the Serial Monitor.

## Goal
Learn how to measure network latency, evaluate TCP handshake speeds, and diagnose connection health.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Every 5 seconds, it initiates a connection to `google.com` on port 80. It measures the duration of the handshake process in microseconds and prints the latency in milliseconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Ping Request tool (TCP Handshake latency analyzer)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Target host to ping (must have port 80 open)
const char* pingHost = "google.com";
const int pingPort = 80;

void performTCPPing() {
  WiFiClient client;
  
  Serial.print("Pinging host: ");
  Serial.print(pingHost);
  Serial.print("...");
  
  // 1. Record start timestamp in microseconds
  unsigned long startTimeUs = micros();
  
  // 2. Attempt TCP handshake connection
  bool connected = client.connect(pingHost, pingPort);
  
  // 3. Record stop timestamp
  unsigned long endTimeUs = micros();
  
  if (connected) {
    // Calculate elapsed time in milliseconds
    float latencyMs = (float)(endTimeUs - startTimeUs) / 1000.0;
    
    Serial.print(" Connection successful! Latency: ");
    Serial.print(latencyMs, 2);
    Serial.println(" ms");
    
    // Close the socket connection
    client.stop();
  } else {
    Serial.println(" Ping request failed! Host unreachable.");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 TCP Ping Tool Active");
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
    performTCPPing();
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Ping every 5 seconds
  delay(5000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the ping latency logs print.

## Expected Output
Serial Monitor:
```
ESP32 TCP Ping Tool Active
....
WiFi Connected successfully.
Pinging host: google.com... Connection successful! Latency: 24.52 ms
Pinging host: google.com... Connection successful! Latency: 22.18 ms
```

## Expected Canvas Behavior
* The Serial Monitor logs the connection handshake latency in milliseconds every 5 seconds.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `micros()` | Reads the internal system clock in microseconds for high-resolution timing. |
| `client.connect(...)` | Initiates the TCP connection handshake to establish the link. |
| `endTimeUs - startTimeUs` | Calculates the total time taken to complete the handshake in microseconds. |

## Hardware & Safety Concept: Network Latency and Timeout Configurations
Latency measures the round-trip time (RTT) for a packet to travel from the ESP32 to the target server and back. High latency (e.g. > 200 ms) indicates congestion or a weak WiFi signal. When writing network code, always configure a non-blocking timeout limit to prevent the device from hanging if a packet is lost.

## Try This! (Challenges)
1. **OLED Latency Dashboard**: Add an OLED screen (Project 60) and display the current latency.
2. **Ping average calculator**: Ping the host 5 times and print the average latency, filtering out any anomalies.
3. **Low signal alert**: Sound a buzzer (GPIO 15) if the latency exceeds 150 ms, warning the user of potential lag.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Ping always fails | Port blocked | Confirm that the target server is online and port 80 is open to connections |
| Latency is extremely high | Network congestion | Check for other active devices on the network or relocate the ESP32 closer to the router |
| The ESP32 hangs | Blocking calls | Ensure the socket client is closed using `client.stop()` after each ping |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 TCP Client connection test](16-esp32-tcp-client-connection-test.md)
- [04 - ESP32 WiFi RSSI Signal strength logging](04-esp32-wifi-rssi-signal-strength-logging.md)
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
