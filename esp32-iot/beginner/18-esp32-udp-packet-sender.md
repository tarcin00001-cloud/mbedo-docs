# 18 - ESP32 UDP Packet Sender

Configure the ESP32 to establish a connectionless User Datagram Protocol (UDP) packet link, read telemetry data from a potentiometer, and transmit UDP packets to a remote listener on port 9000.

## Goal
Learn how to use the `WiFiUDP` library, format datagram payloads, transmit connectionless packets, and minimize transport latency.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A potentiometer is on GPIO 34. Every 2 seconds, the ESP32 reads the potentiometer level, packages it into a text payload, and transmits a UDP packet to the target IP address `10.10.0.1` on port 9000.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | Telemetry signal |
| Shared power | VCC / GND | 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Connect the potentiometer wiper to GPIO 34.

## Code
```cpp
// UDP Packet Sender (Lightweight Telemetry Transmitter)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

// Target UDP Listener network details
const char* targetIP = "10.10.0.1"; // Replace with your target computer's IP
const int targetPort = 9000;

WiFiUDP udp;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 UDP Telemetry Sender Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // 1. Initialize local UDP port
  udp.begin(8888); // Local port to listen/bind
  
  Serial.print("Local UDP Port: 8888 | Target: ");
  Serial.print(targetIP);
  Serial.print(":");
  Serial.println(targetPort);
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // Read potentiometer value
    int sensorVal = analogRead(POT_PIN);
    
    // 2. Prepare payload string
    String payload = "POT_VAL:" + String(sensorVal) + ",UPTIME_S:" + String(millis() / 1000);
    
    Serial.print("Transmitting UDP Packet: ");
    Serial.println(payload);
    
    // 3. Begin UDP packet creation
    udp.beginPacket(targetIP, targetPort);
    
    // 4. Write data to packet buffer
    udp.print(payload);
    
    // 5. Transmit packet over the network
    bool success = udp.endPacket();
    
    if (success) {
      Serial.println("Packet transmitted successfully.");
    } else {
      Serial.println("Packet transmission failed!");
    }
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Transmit every 2 seconds
  delay(2000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Potentiometer** onto the canvas.
2. Wire Potentiometer to **GPIO34**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Watch the transmitted values update in the Serial Monitor.
5. On hardware, run a UDP listener on your computer (e.g. `nc -ul 9000`) and verify incoming packets.

## Expected Output
Serial Monitor:
```
==================================
ESP32 UDP Telemetry Sender Setup
==================================
WiFi Connected successfully.
Local UDP Port: 8888 | Target: 10.10.0.1:9000
==================================

Transmitting UDP Packet: POT_VAL:2048,UPTIME_S:2
Packet transmitted successfully.
Transmitting UDP Packet: POT_VAL:3250,UPTIME_S:4
Packet transmitted successfully.
```

## Expected Canvas Behavior
* The Serial Monitor logs the raw telemetry strings and packet status every 2 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFiUDP` | Instantiates a UDP socket utility helper object. |
| `udp.beginPacket(...)` | Starts a UDP packet, addressing it to the target IP and port. |
| `udp.print(payload)` | Writes the string payload into the packet buffer. |
| `udp.endPacket()` | Transmits the completed UDP packet. |

## Hardware & Safety Concept: UDP vs TCP in IoT Projects
**UDP (User Datagram Protocol)** is a connectionless protocol. It does not establish a handshake or confirm packet delivery. This reduces network overhead and transmission latency, making it ideal for streaming high-frequency data (like sensor streams or joystick angles). If packets are dropped due to noise, the system simply waits for the next update. For critical commands (like unlocking a vault), use TCP to ensure delivery.

## Try This! (Challenges)
1. **OLED Status display**: Add an OLED screen (Project 60) and display the count of transmitted packets.
2. **Dynamic Target Config**: Use serial commands to dynamically update the target IP address.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 4) whenever a packet transmission fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Listener does not receive packets | Firewall block | Ensure your computer's firewall is configured to allow incoming traffic on port 9000 |
| Packets are lost on busy networks | Packet size exceeds MTU | Keep UDP payloads under 1400 bytes to avoid network fragmentation |
| `endPacket()` returns false | Router routing error | Verify that the target IP address matches your computer's active IP on the network |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 TCP Client connection test](16-esp32-tcp-client-connection-test.md)
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [27 - ESP32 UDP Joysticks telemetry stream](27-esp32-udp-joysticks-telemetry-stream.md)
