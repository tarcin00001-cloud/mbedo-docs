# 21 - ESP32 UDP Control Switch

Build a remote device control station on the ESP32 that binds to UDP port 9000, listens for packet command tokens ("RELAY_ON", "RELAY_OFF"), drives a relay to actuate connected loads, and returns status confirmations.

## Goal
Learn how to parse packet command strings, control actuators based on wireless inputs, sound confirm chimes, and send UDP status replies.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and binds to UDP port 9000. A relay is on GPIO 13, and a buzzer on GPIO 4. Sending the UDP command `RELAY_ON` closes the relay contacts and sounds the buzzer. Sending `RELAY_OFF` opens the contacts. The ESP32 returns a status packet to confirm the action.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load controller switch |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Confirmation chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the relay from the 5V Vin rail.

## Code
```cpp
// UDP Control Switch (Remote control relay + confirmation beeps)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RELAY_PIN = 13;
const int BUZZER_PIN = 4;
const int localPort = 9000;

WiFiUDP udp;
char packetBuffer[255];

void playChime(bool success) {
  if (success) {
    // Double quick beep
    digitalWrite(BUZZER_PIN, HIGH); delay(60);
    digitalWrite(BUZZER_PIN, LOW);  delay(40);
    digitalWrite(BUZZER_PIN, HIGH); delay(60);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    // Single long buzz
    digitalWrite(BUZZER_PIN, HIGH); delay(400);
    digitalWrite(BUZZER_PIN, LOW);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 UDP Control Switch Starting");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  udp.begin(localPort);
  
  Serial.print("UDP Switch active. IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(localPort);
  Serial.println("==================================\n");
}

void loop() {
  // 1. Check for incoming command packets
  int packetSize = udp.parsePacket();
  
  if (packetSize > 0) {
    // 2. Read packet data
    int len = udp.read(packetBuffer, 255);
    
    if (len > 0) {
      packetBuffer[len] = 0; // Null-terminate string
    }
    
    String cmd = String(packetBuffer);
    cmd.trim(); // Clean trailing whitespace / newlines
    
    Serial.print("[Control Signal] Received: ");
    Serial.print(cmd);
    Serial.print(" | From: ");
    Serial.println(udp.remoteIP());
    
    // 3. Command Evaluator
    if (cmd == "RELAY_ON") {
      digitalWrite(RELAY_PIN, HIGH);
      Serial.println("Contactor closed (LOAD ON).");
      playChime(true);
      
      // 4. Return confirmation packet
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("RELAY STATE: ON");
      udp.endPacket();
    } 
    else if (cmd == "RELAY_OFF") {
      digitalWrite(RELAY_PIN, LOW);
      Serial.println("Contactor opened (LOAD OFF).");
      playChime(true);
      
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("RELAY STATE: OFF");
      udp.endPacket();
    } 
    else {
      // Unknown command error handling
      Serial.println("Error: Unknown Command!");
      playChime(false);
      
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ERROR: UNKNOWN COMMAND");
      udp.endPacket();
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, and **Buzzer** onto the canvas.
2. Wire Relay to **GPIO13** and Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. In simulation, send the UDP command `RELAY_ON` on port 9000. Watch the relay turn ON (green) and the buzzer beep.
5. Send the UDP command `RELAY_OFF`. Watch the relay turn OFF (grey).

## Expected Output
Serial Monitor:
```
==================================
ESP32 UDP Control Switch Starting
==================================
WiFi Connected.
UDP Switch active. IP: 10.10.0.3 | Port: 9000
==================================

[Control Signal] Received: RELAY_ON | From: 10.10.0.5
Contactor closed (LOAD ON).
[Control Signal] Received: RELAY_OFF | From: 10.10.0.5
Contactor opened (LOAD OFF).
```

## Expected Canvas Behavior
* Sending `RELAY_ON` turns the relay widget ON (green) and pulses the buzzer widget green.
* Sending `RELAY_OFF` turns the relay widget OFF (grey).
* Sending invalid commands pulses the buzzer widget green for a longer duration (warning buzz).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `cmd.trim()` | Cleans the command string, removing newlines or carriage return characters. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contacts, powering the connected load. |
| `udp.beginPacket(...)` | Addresses the confirmation packet back to the sender's IP and port. |

## Hardware & Safety Concept: Actuator Control over UDP Sockets
Using raw UDP to control physical actuators (like relays or motor switches) requires careful validation. Because UDP is connectionless and does not verify the sender, anyone on the network can send control packets. To protect the system:
1. **Verification**: Always return a status confirmation packet so the controller can audit state changes.
2. **Hysteresis**: Enforce a minimal delay between switching commands to prevent contact chatter.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the active relay status.
2. **Access Token authentication**: Require commands to include a security prefix (e.g. `KEY:1234,RELAY_ON`).
3. **Contactor trip safety**: Automatically turn the relay OFF if a safety sensor (like a limit switch) is triggered.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay does not switch | Pin assignment conflict | Verify that the relay signal line is wired to GPIO 13 |
| Buzzer sounds error buzz for valid commands | Hidden characters | Ensure the sending utility does not append hidden newlines; use `cmd.trim()` to clean the string |
| The ESP32 does not receive packets | Port blocked | Confirm that the sending device addresses packets to port 9000 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [20 - ESP32 UDP Broadcast Alert](20-esp32-udp-broadcast-alert.md)
- [25 - ESP32 Wireless buzzer receiver](25-esp32-wireless-buzzer-receiver.md)
