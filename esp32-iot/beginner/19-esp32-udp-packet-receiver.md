# 19 - ESP32 UDP Packet Receiver

Configure the ESP32 to bind to a local UDP port, parse incoming datagram packets, toggle a status LED when receiving a "TOGGLE" command, and transmit a confirmation reply to the sender.

## Goal
Learn how to bind UDP sockets to local ports, parse incoming packet structures, extract sender coordinates, and send replies.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and binds to local UDP port 9000. An LED is on GPIO 13. When you send a UDP packet containing `TOGGLE` to the ESP32 on port 9000, it toggles the LED, logs the message, and transmits a reply packet containing `LED TOGGLED!` back to you.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Green LED | Anode (+) | GPIO13 via 330 Ω | Green | Controller status indicator |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Green LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// UDP Packet Receiver (Parse datagrams + remote control)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;
const int localPort = 9000; // Local port to listen on

WiFiUDP udp;
char packetBuffer[255]; // Buffer to hold incoming packet bytes

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 UDP Telemetry Receiver Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 1. Bind UDP socket to local port
  udp.begin(localPort);
  
  Serial.print("UDP socket active. Listening on IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(localPort);
  Serial.println("==================================\n");
}

void loop() {
  // 2. Parse incoming UDP packets
  int packetSize = udp.parsePacket();
  
  if (packetSize > 0) {
    // 3. Read packet bytes into buffer
    // Returns number of bytes read
    int len = udp.read(packetBuffer, 255); 
    
    if (len > 0) {
      packetBuffer[len] = 0; // Null-terminate string
    }
    
    String receivedStr = String(packetBuffer);
    receivedStr.trim(); // Trim spaces/newlines
    
    Serial.print("[UDP Recv] Size: ");
    Serial.print(packetSize);
    Serial.print(" bytes | From: ");
    Serial.print(udp.remoteIP());
    Serial.print(":");
    Serial.print(udp.remotePort());
    Serial.print(" | Payload: ");
    Serial.println(receivedStr);
    
    // 4. Command Parser
    if (receivedStr == "TOGGLE") {
      bool ledState = !digitalRead(LED_PIN);
      digitalWrite(LED_PIN, ledState);
      Serial.print("LED Toggled: "); Serial.println(ledState ? "ON" : "OFF");
      
      // 5. Transmit reply to sender
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("LED TOGGLED! State: ");
      udp.print(ledState ? "ON" : "OFF");
      udp.endPacket();
    } else {
      // General reply
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ACK: Received: " + receivedStr);
      udp.endPacket();
    }
  }
  
  delay(10); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Inspect the Serial Monitor. Note the IP address.
5. On hardware, send a UDP packet containing `TOGGLE` (e.g. `echo -n "TOGGLE" | nc -u 10.10.0.3 9000`). Watch the LED toggle.

## Expected Output
Serial Monitor:
```
==================================
ESP32 UDP Telemetry Receiver Setup
==================================
WiFi Connected.
UDP socket active. Listening on IP: 10.10.0.3 | Port: 9000
==================================

[UDP Recv] Size: 6 bytes | From: 10.10.0.5:54920 | Payload: TOGGLE
LED Toggled: ON
[UDP Recv] Size: 6 bytes | From: 10.10.0.5:54920 | Payload: TOGGLE
LED Toggled: OFF
```

## Expected Canvas Behavior
* Sending `TOGGLE` over the simulated UDP terminal toggles the Green LED widget.
* The Serial Monitor logs the packet sender IP, port, and size.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `udp.begin(localPort)` | Binds the UDP socket to the specified port to listen for incoming datagrams. |
| `udp.parsePacket()` | Checks if a new packet is available in the network buffer. |
| `udp.read(...)` | Reads the packet data into the character buffer. |
| `udp.remoteIP()` | Retrieves the IP address of the sender, allowing addressing replies. |

## Hardware & Safety Concept: Command Parsing Security in UDP Sockets
UDP does not verify the sender's identity. If a system accepts critical commands (like opening a gate) over raw UDP, an attacker can transmit fake packets (spoofing) to control the device. To secure UDP control links:
1. **Authentication**: Include a pre-shared security token in the packet payload.
2. **Filtering**: Restrict connection routing by verifying `udp.remoteIP()` against a list of authorized IP addresses.

## Try This! (Challenges)
1. **OLED Log Dashboard**: Add an OLED screen (Project 60) and display the payload and sender IP.
2. **Buzzer confirmation**: Sound a quick beep on a buzzer (GPIO 4) when a valid command is parsed.
3. **Speed Controller command**: Send speed values (e.g. `SPEED:180`) to control a motor's speed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `parsePacket()` always returns 0 | Port conflict | Verify that no other service on the network is using port 9000 |
| Received text contains strange characters | Null-terminator missing | Ensure `packetBuffer[len] = 0` is executed to end the string |
| Replies fail to reach sender | Port closed | Verify that the sending utility keeps its local UDP port open to receive replies |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [21 - ESP32 UDP Control Switch](21-esp32-udp-control-switch.md)
- [25 - ESP32 Wireless buzzer receiver](25-esp32-wireless-buzzer-receiver.md)
