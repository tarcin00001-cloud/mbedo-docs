# 26 - ESP32 TCP Telemetry Broadcaster

Build a multi-client telemetry streaming station on the ESP32 that hosts a TCP server on port 8080, accepts up to four concurrent socket connections, reads an analog potentiometer sensor, and broadcasts real-time telemetry packets to all connected clients.

## Goal
Learn how to manage multiple active client sockets, format streaming data payloads, broadcast packets over active streams, and handle client disconnections.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a TCP server on port 8080. A potentiometer is on GPIO 34. Up to four clients can connect simultaneously (e.g. using Telnet or Netcat). Every second, the ESP32 reads the sensor and broadcasts the telemetry string to all connected clients.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Analog sensor signal |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// TCP Telemetry Broadcaster (Multi-Client Sensor Streamer)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;
const int serverPort = 8080;

WiFiServer server(serverPort);

// Keep track of connected clients
const int MAX_CLIENTS = 4;
WiFiClient clients[MAX_CLIENTS];

unsigned long lastBroadcastTime = 0;
const unsigned long BROADCAST_INTERVAL_MS = 1000; // Stream data every 1 second

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 TCP Telemetry Broadcaster Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Start TCP Server
  server.begin();
  
  Serial.print("Broadcast Server active. IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(serverPort);
  Serial.println("==================================\n");
}

void loop() {
  // 1. Check for new incoming client connections
  WiFiClient newClient = server.available();
  
  if (newClient) {
    bool slotFound = false;
    
    // Find an empty slot in the client array
    for (int i = 0; i < MAX_CLIENTS; i++) {
      if (!clients[i] || !clients[i].connected()) {
        // Free client resource if disconnected
        if (clients[i]) clients[i].stop();
        
        clients[i] = newClient;
        Serial.print("[Server Info] New client connected on slot: ");
        Serial.println(i);
        
        // Welcome message
        clients[i].println("--- ESP32 Telemetry Live Broadcast ---");
        
        slotFound = true;
        break;
      }
    }
    
    // If server is full, reject connection
    if (!slotFound) {
      Serial.println("[Server Alert] Connection rejected: Client limit reached!");
      newClient.println("Error: Server Full! Disconnecting.");
      newClient.stop();
    }
  }
  
  // 2. Broadcast Telemetry periodically (every 1 second)
  unsigned long now = millis();
  if (now - lastBroadcastTime >= BROADCAST_INTERVAL_MS) {
    int sensorVal = analogRead(POT_PIN);
    float voltage = sensorVal * (3.3 / 4095.0);
    
    // Format payload
    String payload = "SENSOR_RAW:" + String(sensorVal) + 
                     ",VOLT:" + String(voltage, 2) + 
                     "V,UPTIME:" + String(now / 1000) + "s";
                     
    Serial.print("Broadcasting to clients: ");
    Serial.println(payload);
    
    // Loop through client array and push telemetry data
    int activeClients = 0;
    for (int i = 0; i < MAX_CLIENTS; i++) {
      if (clients[i] && clients[i].connected()) {
        clients[i].println(payload);
        activeClients++;
      }
    }
    
    Serial.print("[Server Audit] Active Subscribers: ");
    Serial.println(activeClients);
    
    lastBroadcastTime = now;
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. In simulation, connect a TCP client (e.g. using Telnet or Netcat) to the ESP32 IP on port 8080. Verify that sensor values stream in real time.
5. Slide the potentiometer widget. Verify that the streamed value changes.

## Expected Output
Serial Monitor:
```
==================================
ESP32 TCP Telemetry Broadcaster Setup
==================================
WiFi Connected.
Broadcast Server active. IP: 10.10.0.3 | Port: 8080
==================================

[Server Info] New client connected on slot: 0
Broadcasting to clients: SENSOR_RAW:2048,VOLT:1.65V,UPTIME:12s
[Server Audit] Active Subscribers: 1
```

Client Terminal Screen:
```
--- ESP32 Telemetry Live Broadcast ---
SENSOR_RAW:2048,VOLT:1.65V,UPTIME:12s
SENSOR_RAW:2150,VOLT:1.73V,UPTIME:13s
```

## Expected Canvas Behavior
* The Serial Monitor logs the telemetry strings and subscriber counts every second.
* Adjusting the potentiometer updates the values streamed to all active client consoles.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFiClient clients[MAX_CLIENTS]` | Allocates an array to track up to 4 concurrent client connections. |
| `!clients[i] || !clients[i].connected()` | Identifies inactive client slots to assign new connections. |
| `clients[i].println(payload)` | Transmits the data payload to each connected client over TCP. |

## Hardware & Safety Concept: Managed Connection Sockets in IoT Servers
A microcontroller has limited memory and processing resources compared to a computer. If a TCP server allows unlimited connections, it will exhaust its socket pool and heap memory, causing the ESP32 to crash. To protect the system, IoT servers must enforce connection limits, reject client requests when the pool is full, and clean up disconnected client structures.

## Try This! (Challenges)
1. **OLED Subscriber HUD**: Add an OLED screen (Project 60) and display the count of active subscribers and the current sensor reading.
2. **Dynamic Client command**: Allow connected clients to send commands (like `LED_ON`) to interact with the ESP32.
3. **Buzzer alert on join**: Sound a quick beep on a buzzer (GPIO 4) when a client joins or leaves.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Server rejects new connections | Connection slot leak | Verify that disconnected clients are cleaned up and `clients[i].stop()` is called |
| Telemetry stream is sluggish | Buffer congestion | Limit the stream frequency to 1 Hz or 2 Hz to avoid overloading the network buffers |
| Client disconnects instantly | Remote client closed socket | Check the client utility settings, ensuring the connection stays active |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [17 - ESP32 TCP Server setup](17-esp32-tcp-server-setup.md)
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [73 - ESP32 WebSocket Sensor Streamer](../../intermediate/73-esp32-websocket-sensor-streamer.md)
