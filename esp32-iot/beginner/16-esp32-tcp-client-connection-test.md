# 16 - ESP32 TCP Client Connection Test

Configure the ESP32 to establish a raw TCP socket connection to a remote server, transmit an HTTP request packet, and log the incoming response stream to the Serial Monitor.

## Goal
Learn how to use the `WiFiClient` library, establish TCP socket handshakes, transmit raw protocol request blocks, and parse stream responses.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Once connected, it initiates a raw TCP connection to the test server `httpbin.org` on port 80 every 10 seconds, transmits a basic HTTP GET request, and logs the response stream to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// TCP Client connection test (Raw TCP Socket transmission)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Target TCP Server details
const char* serverHost = "httpbin.org";
const int serverPort = 80;

void testTCPConnection() {
  // 1. Instantiate the WiFiClient socket object
  WiFiClient client;
  
  Serial.print("Connecting to TCP Server: ");
  Serial.print(serverHost);
  Serial.print(" on port ");
  Serial.println(serverPort);
  
  // 2. Establish TCP handshake connection
  if (!client.connect(serverHost, serverPort)) {
    Serial.println("TCP Connection failed!");
    return;
  }
  
  Serial.println("Connected to server successfully. Transmitting request...");
  
  // 3. Transmit raw HTTP GET header request over the TCP socket
  client.println("GET /ip HTTP/1.1");
  client.println("Host: httpbin.org");
  client.println("Connection: close");
  client.println(); // CR LF blank line ends HTTP header request
  
  // 4. Wait for and read the response stream
  unsigned long timeout = millis();
  while (client.available() == 0) {
    if (millis() - timeout > 5000) {
      Serial.println(">>> TCP Response Timeout! <<<");
      client.stop();
      return;
    }
    delay(10);
  }
  
  Serial.println("--- Response Received ---");
  // Read and print response characters until stream is empty
  while (client.available()) {
    char ch = client.read();
    Serial.write(ch);
  }
  Serial.println("\n-------------------------");
  
  // 5. Close the socket connection
  client.stop();
  Serial.println("TCP connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing TCP Client Test...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nConnected to network!");
}

void loop() {
  testTCPConnection();
  
  // Test again in 10 seconds
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the server response (containing the ESP32's simulated IP) prints.

## Expected Output
Serial Monitor:
```
Initializing TCP Client Test...
....
Connected to network!
Connecting to TCP Server: httpbin.org on port 80
Connected to server successfully. Transmitting request...
--- Response Received ---
HTTP/1.1 200 OK
Date: Sat, 11 Jul 2026 01:15:00 GMT
Content-Type: application/json
Content-Length: 31
Connection: close

{
  "origin": "10.10.0.3"
}
-------------------------
TCP connection closed.
```

## Expected Canvas Behavior
* The Serial Monitor prints the connection details and the server response every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFiClient` | Instantiates a TCP client socket object. |
| `client.connect(...)` | Initiates the TCP three-way handshake with the target server. |
| `client.println(...)` | Transmits raw text over the established TCP socket. |
| `client.stop()` | Closes the socket connection, releasing memory resources. |

## Hardware & Safety Concept: TCP Sockets vs HTTP Clients
At the core of all web communication is the **TCP (Transmission Control Protocol)** socket. Higher-level protocols (like HTTP, WebSockets, or MQTT) run on top of TCP. Using a raw `WiFiClient` socket allows transmitting custom protocols, bypassing the overhead of full libraries. When opening sockets, always close them using `client.stop()` to prevent socket leaks that exhaust the ESP32's memory resources.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display the TCP connection state.
2. **Local Echo Client**: Configure the client to connect to a local TCP echo server (Project 17) and verify returned messages.
3. **HTTPS Secure client**: Use `WiFiClientSecure` to establish an encrypted TLS TCP connection on port 443.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails immediately | Server IP/Port blocked | Verify that the target server is online and port 80 is open |
| System hangs on receive | Timeout missing | Implement a non-blocking timeout check to prevent the code from hanging if the server fails to reply |
| Response is garbled | Baud rate mismatch | Ensure the Serial Monitor baud rate matches the sketch setting (115200) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [17 - ESP32 TCP Server setup](17-esp32-tcp-server-setup.md)
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
