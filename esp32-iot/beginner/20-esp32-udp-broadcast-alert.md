# 20 - ESP32 UDP Broadcast Alert

Configure the ESP32 to monitor a push button and transmit network-wide User Datagram Protocol (UDP) broadcast warning packets to all devices on the local subnet when the button is pressed, sounding a buzzer.

## Goal
Learn how to use UDP broadcast addressing, configure network-wide transmissions, debounce inputs, and trigger alarms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A button is on GPIO 12, and a buzzer on GPIO 15. When the button is pressed, the buzzer sounds, the ESP32 addresses a UDP packet to the broadcast IP `255.255.255.255` on port 9000, and sends it, alerting all devices on the subnet.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Orange / Red | Emergency alert button |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Status chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 12.

## Code
```cpp
// UDP Broadcast Alert (Emergency Broadcast Alert system)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUTTON_PIN = 12;
const int BUZZER_PIN = 15;

// Broadcast IP addresses all hosts on the local subnet
IPAddress broadcastIP(255, 255, 255, 255);
const int broadcastPort = 9000;

WiFiUDP udp;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 UDP Broadcast Alert Starting");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(8888); // Bind local port
  Serial.println("Ready. Press button to broadcast alert.");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read Emergency Alert Button
    bool btnState = (digitalRead(BUTTON_PIN) == HIGH);
    
    if (btnState) {
      // 2. Sound local alert chime
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      
      // 3. Format broadcast payload
      String payload = "ALERT_LEVEL:CRITICAL,NODE:ESP32_SECURE_1,UPTIME:" + String(millis() / 1000);
      
      Serial.print("Transmitting Broadcast Packet: ");
      Serial.println(payload);
      
      // 4. Begin UDP broadcast packet
      udp.beginPacket(broadcastIP, broadcastPort);
      
      // 5. Write and transmit
      udp.print(payload);
      bool success = udp.endPacket();
      
      if (success) {
        Serial.println("Broadcast sent successfully.");
      } else {
        Serial.println("Broadcast send failed!");
      }
      
      delay(500); // Debounce / prevent spamming
    }
  } else {
    Serial.println("WiFi Link offline!");
    delay(1000);
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Button**, and **Buzzer** onto the canvas.
2. Wire Button to **GPIO12** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Click the button widget. Watch the buzzer beep.
5. Inspect the Serial Monitor. Verify that the broadcast packet transmits.

## Expected Output
Serial Monitor:
```
==================================
ESP32 UDP Broadcast Alert Starting
==================================
WiFi Connected successfully.
Ready. Press button to broadcast alert.
==================================

Transmitting Broadcast Packet: ALERT_LEVEL:CRITICAL,NODE:ESP32_SECURE_1,UPTIME:12
Broadcast sent successfully.
```

## Expected Canvas Behavior
* Pressing the button widget pulses the buzzer widget green.
* The Serial Monitor logs the broadcast transmissions.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `IPAddress(255, 255, 255, 255)` | Sets the target to the limited broadcast address, targeting all devices on the local subnet. |
| `udp.beginPacket(...)` | Addresses the UDP packet to the broadcast IP and target port. |
| `udp.endPacket()` | Transmits the packet to all network cards on the local subnet. |

## Hardware & Safety Concept: Network Broadcasting and Port Selection
A **UDP Broadcast** allows sending a single packet that is received by all devices on the local subnet. This is useful for emergency alerts or device discovery (finding the IP address of an ESP32 on a network). Because routers drop broadcast packets by default to prevent congestion, these transmissions do not cross subnet boundaries, restricting them to the local network.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a warning screen when a broadcast is sent.
2. **Dynamic Alert sound**: Modulate the buzzer frequency based on how long the button is held.
3. **Directed Subnet broadcast**: Configure the broadcast address to target a specific subnet range (e.g. `192.168.1.255`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Broadcast fails to transmit | WiFi offline | Verify that the ESP32 is connected to the network before sending |
| Receivers ignore the packet | Port mismatch | Ensure the receiving devices are listening on port 9000 |
| Router drops the broadcast | AP isolation enabled | Disable "Client Isolation" or "AP Isolation" in the router configuration to allow local communication |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [21 - ESP32 UDP Control Switch](21-esp32-udp-control-switch.md)
