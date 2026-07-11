# 30 - ESP32 Laser Receiver Wireless Gate Trigger

Build a laser break-beam gate trigger on the ESP32 that samples an analog photoresistor (LDR) to detect when a laser beam is broken, sounds a local alert beep, and transmits a wireless UDP warning packet.

## Goal
Learn how to implement threshold checks on analog light sensors, manage transmit cooldown timers, format alert packets, and build laser break-beam gates.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LDR (photoresistor) is on GPIO 34, and a buzzer on GPIO 15. A laser pointer shines on the LDR. When a person walks through and breaks the beam (causing the LDR reading to rise), the buzzer sounds, and the ESP32 transmits a UDP packet containing `BEAM_BROKEN` to the broadcast IP `255.255.255.255` on port 9000.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Photoresistor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | Light sensor analog input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | Black | Divider pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning chime |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The photoresistor requires a 10 kΩ pull-down resistor to ground to form a voltage divider. This maps dark to low voltage (low raw ADC values) and light to high voltage (high raw ADC values).

## Code
```cpp
// Laser receiver wireless gate trigger (Laser break-beam alarm)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;
const int BUZZER_PIN = 15;

// Beam threshold configuration
// Light shining (laser ON) -> high reading (e.g. > 2500)
// Beam broken (laser OFF)   -> low reading  (e.g. < 1000)
const int BEAM_THRESHOLD = 1500;

// Security controller target details (using broadcast)
IPAddress targetIP(255, 255, 255, 255);
const int targetPort = 9000;

WiFiUDP udp;

// Cooldown variables
unsigned long lastTriggerTime = 0;
const unsigned long TRIGGER_COOLDOWN_MS = 3000; // 3-second cooldown

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LDR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Laser Gate Trigger Starting");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(8888); // Bind local port
  Serial.println("Laser gate alignment active.");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read LDR Analog value (0 to 4095)
    int ldrRaw = analogRead(LDR_PIN);
    unsigned long now = millis();
    
    // 2. Evaluate Beam status
    bool beamBroken = (ldrRaw < BEAM_THRESHOLD);
    
    // Log alignment values
    static unsigned long lastLog = 0;
    if (now - lastLog >= 1000) {
      Serial.print("LDR light level: ");
      Serial.print(ldrRaw);
      Serial.println(beamBroken ? " | BEAM STATUS: BROKEN" : " | BEAM STATUS: ALIGNED");
      lastLog = now;
    }
    
    // 3. Trigger alert with cooldown
    if (beamBroken) {
      if (now - lastTriggerTime >= TRIGGER_COOLDOWN_MS) {
        Serial.println("!!! ALERT: Laser beam broken! Gate triggered. !!!");
        
        // Sound local buzzer alert chime
        digitalWrite(BUZZER_PIN, HIGH);
        delay(150);
        digitalWrite(BUZZER_PIN, LOW);
        
        // 4. Format and transmit UDP alert packet
        String payload = "GATE_ALERT:BEAM_BROKEN,LDR_VAL:" + String(ldrRaw);
        
        udp.beginPacket(targetIP, targetPort);
        udp.print(payload);
        udp.endPacket();
        
        lastTriggerTime = now;
      }
    }
  } else {
    Serial.println("WiFi offline!");
    delay(1000);
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LDR**, and **Buzzer** onto the canvas.
2. Wire LDR to **GPIO34** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Adjust the LDR light level slider below 1500. Watch the buzzer beep.
5. Inspect the Serial Monitor. Verify that the broadcast packet transmits.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Laser Gate Trigger Starting
==================================
WiFi Connected successfully.
Laser gate alignment active.
==================================

LDR light level: 3200 | BEAM STATUS: ALIGNED
LDR light level: 3150 | BEAM STATUS: ALIGNED
!!! ALERT: Laser beam broken! Gate triggered. !!!
LDR light level: 450 | BEAM STATUS: BROKEN
```

## Expected Canvas Behavior
* Lowering the LDR slider value below 1500 pulses the buzzer widget green.
* The Serial Monitor logs the LDR readings and triggers the UDP broadcast alert.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ldrRaw < BEAM_THRESHOLD` | Evaluates if the light level has dropped, indicating the laser beam is blocked. |
| `now - lastTriggerTime >= 3000` | Enforces a 3-second cooldown to prevent spamming packets while the beam is blocked. |
| `udp.beginPacket(...)` | Addresses the alert packet to the broadcast IP on port 9000. |

## Hardware & Safety Concept: Laser Break-beam Gates and Cooldown Timers
A laser break-beam system aligns a narrow laser beam with a light sensor (LDR). When aligned, the sensor reads high light levels. If a person blocks the beam, the sensor reading drops. To prevent flooding the network with packets when a person stands in the gate, the controller must implement a **cooldown timer**, blocking subsequent packets for a short duration.

## Try This! (Challenges)
1. **OLED Alignment HUD**: Add an OLED screen (Project 60) and display a bar graph showing the laser alignment level.
2. **Evacuation Siren integration**: Sound a continuous siren (Project 29) if the beam remains broken for more than 5 seconds.
3. **RSSI Signal meter**: Check signal strength (RSSI) (Project 04) and flash an LED if the connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The gate triggers continuously when laser is ON | Alignment threshold incorrect | Read the raw LDR values when the laser is on and set the threshold below that level |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 15 |
| Alerts are spammed when blocked | Cooldown timer logic error | Verify that the cooldown check compares `now - lastTriggerTime` against the interval |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [20 - ESP32 UDP Broadcast Alert](20-esp32-udp-broadcast-alert.md)
- [141 - ESP32 Sounding Sentry Guard](../expert/141-esp32-sounding-sentry-guard.md)
