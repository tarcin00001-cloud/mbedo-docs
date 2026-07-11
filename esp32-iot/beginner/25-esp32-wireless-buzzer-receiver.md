# 25 - ESP32 Wireless Buzzer Receiver

Build a wireless alert receiver node on the ESP32 that binds to UDP port 9000, parses incoming alert command tokens ("BEEP_ONCE", "BEEP_SIREN"), drives a buzzer and a status LED to trigger alarm sequences, and returns status confirmations.

## Goal
Learn how to parse alert commands, trigger distinct auditory and visual warning sequences, send confirmation packets, and build remote alarms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and binds to UDP port 9000. An active buzzer is on GPIO 4, and a Red LED on GPIO 12. Sending the UDP command `BEEP_ONCE` sounds a single beep and flashes the LED. Sending `BEEP_SIREN` triggers a double chirp pattern. The ESP32 returns a status packet to confirm the alert has been sounded.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Alarm / alert buzzer |
| Red LED | Anode (+) | GPIO12 via 330 Ω | Red | Warning status indicator |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 12 via a 330 Ω resistor.

## Code
```cpp
// Wireless buzzer receiver (Remote alarm buzzer node)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 4;
const int LED_PIN = 12;
const int localPort = 9000;

WiFiUDP udp;
char packetBuffer[255];

void triggerAlert(int beepCount, int beepDurationMs, int gapMs) {
  for (int i = 0; i < beepCount; i++) {
    // Sound buzzer and light LED
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(LED_PIN, HIGH);
    delay(beepDurationMs);
    
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(LED_PIN, LOW);
    
    if (i < beepCount - 1) {
      delay(gapMs);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Wireless Buzzer Alert Node");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(localPort);
  
  Serial.print("Buzzer Node active. Listening on IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(localPort);
  Serial.println("==================================\n");
}

void loop() {
  // 1. Check for incoming alert command packets
  int packetSize = udp.parsePacket();
  
  if (packetSize > 0) {
    // 2. Read packet data
    int len = udp.read(packetBuffer, 255);
    
    if (len > 0) {
      packetBuffer[len] = 0; // Null-terminate string
    }
    
    String cmd = String(packetBuffer);
    cmd.trim(); // Clean trailing spaces/newlines
    
    Serial.print("[Alert Signal] Received: ");
    Serial.print(cmd);
    Serial.print(" | From: ");
    Serial.println(udp.remoteIP());
    
    // 3. Command Evaluator
    if (cmd == "BEEP_ONCE") {
      Serial.println("Sounding single alert beep.");
      triggerAlert(1, 150, 0);
      
      // 4. Return confirmation packet
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ACK: BEEPED ONCE");
      udp.endPacket();
    } 
    else if (cmd == "BEEP_SIREN") {
      Serial.println("Sounding double warning chime.");
      triggerAlert(2, 80, 80);
      
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ACK: SIREN TRIGGERED");
      udp.endPacket();
    } 
    else if (cmd == "SILENT") {
      Serial.println("System Silenced.");
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(LED_PIN, LOW);
      
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ACK: SILENCED");
      udp.endPacket();
    } 
    else {
      Serial.println("Error: Unknown Command!");
      triggerAlert(1, 400, 0); // Sound long error tone
      
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ERROR: UNKNOWN COMMAND");
      udp.endPacket();
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Buzzer**, and **LED** onto the canvas.
2. Wire Buzzer to **GPIO4** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, send the UDP command `BEEP_ONCE` on port 9000. Watch the buzzer beep and the LED flash.
5. Send the UDP command `BEEP_SIREN`. Watch the double flash/beep pattern.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Wireless Buzzer Alert Node
==================================
WiFi Connected successfully.
Buzzer Node active. Listening on IP: 10.10.0.3 | Port: 9000
==================================

[Alert Signal] Received: BEEP_ONCE | From: 10.10.0.5
Sounding single alert beep.
[Alert Signal] Received: BEEP_SIREN | From: 10.10.0.5
Sounding double warning chime.
```

## Expected Canvas Behavior
* Sending `BEEP_ONCE` pulses the buzzer and Red LED widgets once.
* Sending `BEEP_SIREN` pulses the buzzer and Red LED widgets twice in rapid succession.
* Sending invalid commands pulses the buzzer widget green for a longer duration (error tone).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `triggerAlert(...)` | Drives the buzzer and status LED pins in sync to generate the alert sequence. |
| `udp.begin(localPort)` | Binds the UDP socket to the specified port to listen for incoming command packets. |
| `udp.beginPacket(...)` | Addresses the confirmation packet back to the sender's IP and port. |

## Hardware & Safety Concept: Wireless Warning Systems and Latency
Emergency warning systems (like fire alarms or evacuation sirens) must trigger quickly once a command is sent. Because TCP requires establishing a connection handshake and verifying packet receipt, it can introduce delays on busy networks. **UDP** provides low-latency transmissions, ensuring the alarm sounds immediately.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning screen when an alert is active.
2. **Interactive Silence Button**: Add a button on GPIO 13 that allows users to manually silence the active alarm.
3. **RSSI Signal meter**: Check signal strength (RSSI) (Project 04) and flash the LED if the connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 4 |
| LED stays ON | Loop timing issue | Verify that the `triggerAlert` function turns the LED pin LOW after each beep |
| The ESP32 does not receive packets | Port blocked | Confirm that the sending device addresses packets to port 9000 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [21 - ESP32 UDP Control Switch](21-esp32-udp-control-switch.md)
- [24 - ESP32 Wireless button trigger](24-esp32-wireless-button-trigger.md)
