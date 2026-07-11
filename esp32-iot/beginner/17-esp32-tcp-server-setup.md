# 17 - ESP32 TCP Server Setup

Configure the ESP32 to host a local raw TCP socket server on port 8080, listen for incoming client connections, read incoming data packets, and echo the bytes directly back to the sender.

## Goal
Learn how to use the `WiFiServer` library, listen on custom TCP ports, accept client socket connections, and build an echo service.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a TCP server listening on port 8080. Using a TCP terminal client (like Netcat or Telnet), you can connect to the ESP32's IP. Any characters you type will be sent to the ESP32, printed on the Serial Monitor, and echoed back to your terminal screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// TCP Server setup (Echo incoming packets)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Host TCP Server on port 8080
const int serverPort = 8080;
WiFiServer server(serverPort);

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Raw TCP Echo Server Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 1. Initialize and start the TCP Server
  server.begin();
  
  Serial.print("TCP Server active and listening on IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(serverPort);
  Serial.println("==================================\n");
}

void loop() {
  // 2. Check for incoming client connections
  WiFiClient client = server.available();
  
  if (client) {
    Serial.print("[Server Alert] Client connected! IP: ");
    Serial.println(client.remoteIP());
    
    // Send a welcome message to the client
    client.println("--- Welcome to the ESP32 TCP Echo Server ---");
    client.println("Type characters and press Enter to receive the echo.");
    client.println("Send 'close' to disconnect.");
    client.println();
    
    String clientInput = "";
    
    // 3. Monitor active client session
    while (client.connected()) {
      if (client.available()) {
        char ch = client.read();
        
        // Echo character back to the client
        client.write(ch);
        
        // Log character to the Serial Monitor
        if (ch == '\n' || ch == '\r') {
          if (clientInput.length() > 0) {
            Serial.print("[Echo Log] Client typed: ");
            Serial.println(clientInput);
            
            // Check disconnect keyword
            if (clientInput == "close") {
              client.println("Goodbye!");
              break;
            }
            clientInput = ""; // Clear buffer
          }
        } else {
          // Accumulate printable characters
          if (isprint(ch)) {
            clientInput += ch;
          }
        }
      }
      delay(2);
    }
    
    // 4. Terminate client connection
    client.stop();
    Serial.println("[Server Alert] Client disconnected.");
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Note the printed IP address and port (`8080`).
4. On hardware, open a terminal on your computer and run `nc 10.10.0.3 8080` (replacing the IP with the ESP32's IP). Type characters and watch them echo back.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Raw TCP Echo Server Setup
==================================
WiFi Connected.
TCP Server active and listening on IP: 10.10.0.3 | Port: 8080
==================================

[Server Alert] Client connected! IP: 10.10.0.5
[Echo Log] Client typed: Hello ESP32
[Echo Log] Client typed: close
[Server Alert] Client disconnected.
```

## Expected Canvas Behavior
* The Serial Monitor logs client connections and the echoed text input.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFiServer server(8080)` | Instantiates a TCP server object listening on port 8080. |
| `server.begin()` | Starts the server, opening the port for incoming connections. |
| `server.available()` | Checks if a client has connected, returning a socket client object. |
| `client.write(ch)` | Echoes the received character directly back to the client's socket. |

## Hardware & Safety Concept: Raw TCP Servers vs HTTP Web Servers
Raw TCP servers listen for direct socket streams (binary or text). They lack the heavy parsing overhead of HTTP (which requires parsing request headers, methods, and URLs). This makes raw TCP ideal for low-latency machine-to-machine (M2M) communications. To ensure stability, write non-blocking code inside client sessions so the server can accept new connections if a client drops off.

## Try This! (Challenges)
1. **Interactive OLED Echo**: Add an OLED screen (Project 60) and display the text typed by the client.
2. **Dual-client host**: Modify the code to handle multiple concurrent client connections.
3. **LED Remote Toggle**: Toggle a Green LED (GPIO 13) if the client sends the command "led_on" or "led_off".

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Client cannot connect to server | Firewall block | Ensure your computer's firewall allows outgoing traffic on port 8080 |
| Server freezes after client disconnects | Infinite loop in session | Verify that the session loop checks `client.connected()` and breaks if the client drops |
| Text is duplicated | Terminal local echo active | Turn off local echo in your terminal client configuration |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 TCP Client connection test](16-esp32-tcp-client-connection-test.md)
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
