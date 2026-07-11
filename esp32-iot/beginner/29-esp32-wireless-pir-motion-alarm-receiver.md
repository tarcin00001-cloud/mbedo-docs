# 29 - ESP32 Wireless PIR Motion Alarm Receiver

Configure the ESP32 to bind to UDP port 9000, parse incoming motion alert packets ("MOTION_DETECTED") from remote sensor nodes, and trigger a pulsing audio-visual security siren using a buzzer and red LED.

## Goal
Learn how to parse wireless security alerts, manage timed alarm siren states, send confirmation packets, and build remote receivers.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and binds to UDP port 9000. A buzzer is on GPIO 4, and a Red LED on GPIO 12. When a remote sensor node sends a UDP packet containing `MOTION_DETECTED` on port 9000, the ESP32 triggers a pulsing siren (beeping and flashing at 5 Hz) for 5 seconds to alert the user.

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
| Active Buzzer | VCC (+) | GPIO4 | Blue | Alarm siren |
| Red LED | Anode (+) | GPIO12 via 330 Ω | Red | Warning indicator LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 12 via a 330 Ω resistor.

## Code
```cpp
// Wireless PIR motion alarm receiver (Security alarm node)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 4;
const int LED_PIN = 12;
const int localPort = 9000;

WiFiUDP udp;
char packetBuffer[255];

// Alarm state variables
bool alarmActive = false;
unsigned long alarmStartTime = 0;
const unsigned long ALARM_DURATION_MS = 5000; // Siren active for 5 seconds

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Security Alarm Receiver Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(localPort);
  
  Serial.print("Alarm Receiver active. IP: ");
  Serial.print(WiFi.localIP());
  Serial.print(" | Port: ");
  Serial.println(localPort);
  Serial.println("==================================\n");
}

void loop() {
  unsigned long now = millis();
  
  // 1. Check for incoming motion alert packets
  int packetSize = udp.parsePacket();
  
  if (packetSize > 0) {
    // Read packet data
    int len = udp.read(packetBuffer, 255);
    
    if (len > 0) {
      packetBuffer[len] = 0;
    }
    
    String payload = String(packetBuffer);
    payload.trim();
    
    Serial.print("[Security Signal] Received: ");
    Serial.print(payload);
    Serial.print(" | From: ");
    Serial.println(udp.remoteIP());
    
    // 2. Alert Evaluator
    if (payload == "MOTION_DETECTED") {
      Serial.println("!!! INTRUDER WARNING: MOTION DETECTED !!!");
      
      // Trigger Alarm State
      alarmActive = true;
      alarmStartTime = now;
      
      // Return confirmation packet
      udp.beginPacket(udp.remoteIP(), udp.remotePort());
      udp.print("ALARM_ACK: TRIGGERED");
      udp.endPacket();
    }
  }
  
  // 3. Process Active Alarm State (Pulsing Siren)
  if (alarmActive) {
    if (now - alarmStartTime < ALARM_DURATION_MS) {
      // Pulse buzzer and LED at 5 Hz (100 ms ON, 100 ms OFF)
      static unsigned long lastPulse = 0;
      static bool pulseState = LOW;
      
      if (now - lastPulse >= 100) {
        pulseState = !pulseState;
        digitalWrite(BUZZER_PIN, pulseState);
        digitalWrite(LED_PIN, pulseState);
        lastPulse = now;
      }
    } else {
      // Alarm timeout reached: Silence system
      Serial.println("Siren duration complete. Silencing alarm.");
      alarmActive = false;
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(LED_PIN, LOW);
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Buzzer**, and **LED** onto the canvas.
2. Wire Buzzer to **GPIO4** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, send the UDP command `MOTION_DETECTED` on port 9000. Watch the buzzer and Red LED pulse.
5. Wait 5 seconds. The alarm silences automatically.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Security Alarm Receiver Setup
==================================
WiFi Connected successfully.
Alarm Receiver active. IP: 10.10.0.3 | Port: 9000
==================================

[Security Signal] Received: MOTION_DETECTED | From: 10.10.0.5
!!! INTRUDER WARNING: MOTION DETECTED !!!
Siren duration complete. Silencing alarm.
```

## Expected Canvas Behavior
* Sending `MOTION_DETECTED` causes the buzzer and Red LED widgets to pulse green/red rapidly.
* After 5 seconds, the widgets stop pulsing and return to their off state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `now - alarmStartTime < 5000` | Keeps the alarm active for 5 seconds after receiving the trigger. |
| `now - lastPulse >= 100` | Controls the 5 Hz pulse rate (100 ms intervals) for the buzzer and LED. |
| `udp.beginPacket(...)` | Addresses the confirmation packet back to the sender's IP and port. |

## Hardware & Safety Concept: Non-Blocking Timed States in Alarms
Alarm systems must manage timed states (like sounding a siren for 5 seconds) without blocking execution. If the controller uses a blocking `delay(5000)` to sound the siren, it cannot read new sensor inputs or parse incoming packets during that time, leaving the system vulnerable. Implementing non-blocking timers using `millis()` keeps the loop responsive.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a flashing warning message during alerts.
2. **Interactive Silence Button**: Add a button on GPIO 13 to manually silence the active alarm.
3. **RSSI Signal meter**: Check signal strength (RSSI) (Project 04) and flash the LED if the connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound during alarm | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 4 |
| LED stays ON | Loop timing issue | Verify that the code turns the LED and buzzer pins LOW after the alarm state ends |
| The ESP32 does not receive packets | Port blocked | Confirm that the sending device addresses packets to port 9000 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [25 - ESP32 Wireless buzzer receiver](25-esp32-wireless-buzzer-receiver.md)
- [141 - ESP32 Sounding Sentry Guard](../expert/141-esp32-sounding-sentry-guard.md)
