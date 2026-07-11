# 28 - ESP32 Wireless Limit Switch Reporter

Build an industrial safety reporter node on the ESP32 that monitors a physical limit switch (button), detects contact closure/opening events, and transmits wireless UDP status packets to a safety controller.

## Goal
Learn how to implement edge-triggered switch reporting, format safety state packets, sound alert chimes, and minimize transmission latency.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A limit switch (push button) is on GPIO 12, and a buzzer on GPIO 15. When the limit switch closes (contacts touch), the buzzer sounds, and the ESP32 transmits a UDP packet containing `LIMIT_SWITCH:CLOSED` to the broadcast IP `255.255.255.255` on port 9000. When it opens, it transmits `LIMIT_SWITCH:OPEN`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Button (Limit Switch) | `button` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Limit Switch | Pin 1 / Pin 2 | GPIO12 / 3V3 | Orange / Red | Safety switch input |
| Limit Switch | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning alarm |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the limit switch with an external 10 kΩ pull-down resistor to GPIO 12.

## Code
```cpp
// Wireless limit switch reporter (Industrial safety transmitter node)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LIMIT_SW_PIN = 12;
const int BUZZER_PIN = 15;

// Safety Controller target details (using network broadcast)
IPAddress targetIP(255, 255, 255, 255);
const int targetPort = 9000;

WiFiUDP udp;

// State tracking variables
bool lastSwitchState = LOW;

void sendSafetyUpdate(String stateLabel) {
  String payload = "NODE:GATE_CONTROLLER_1,LIMIT_STATE:" + stateLabel;
  
  Serial.print("Transmitting Safety Alert: ");
  Serial.println(payload);
  
  udp.beginPacket(targetIP, targetPort);
  udp.print(payload);
  udp.endPacket();
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LIMIT_SW_PIN, INPUT); // External pull-down wired
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Wireless Limit Switch Reporter");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(8888); // Bind local port
  
  // Read initial state
  lastSwitchState = (digitalRead(LIMIT_SW_PIN) == HIGH);
  sendSafetyUpdate(lastSwitchState ? "CLOSED" : "OPEN");
  
  Serial.println("Safety reporting active.");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read current limit switch state
    bool currentSwitchState = (digitalRead(LIMIT_SW_PIN) == HIGH);
    
    // 2. State transition detection
    if (currentSwitchState != lastSwitchState) {
      if (currentSwitchState == HIGH) {
        // Limit hit: Sound local chime and transmit CLOSED packet
        Serial.println("[Safety Alert] Limit switch hit!");
        digitalWrite(BUZZER_PIN, HIGH); delay(150); digitalWrite(BUZZER_PIN, LOW);
        
        sendSafetyUpdate("CLOSED");
      } else {
        // Limit released: Transmit OPEN packet
        Serial.println("[Safety Alert] Limit switch released.");
        sendSafetyUpdate("OPEN");
      }
      
      lastSwitchState = currentSwitchState;
      delay(100); // Debounce delay
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
4. Click the button widget (switch closed). Watch the buzzer beep and the CLOSED packet transmit.
5. Release the button widget (switch open). Watch the OPEN packet transmit.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Wireless Limit Switch Reporter
==================================
WiFi Connected successfully.
Safety reporting active.
==================================

[Safety Alert] Limit switch hit!
Transmitting Safety Alert: NODE:GATE_CONTROLLER_1,LIMIT_STATE:CLOSED
[Safety Alert] Limit switch released.
Transmitting Safety Alert: NODE:GATE_CONTROLLER_1,LIMIT_STATE:OPEN
```

## Expected Canvas Behavior
* Pressing the button widget pulses the buzzer widget green.
* The Serial Monitor logs the safety alert transmissions.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `currentSwitchState != lastSwitchState` | Detects a change in the switch state to prevent repeatedly sending packets. |
| `sendSafetyUpdate(...)` | Formats and transmits the safety status packet. |
| `udp.beginPacket(...)` | Addresses the packet to the broadcast IP on port 9000. |

## Hardware & Safety Concept: Limit Switches in Automated Systems
Limit switches act as safety triggers in automated systems (such as CNC gantries or automated gates). When a moving carriage hits the limit switch, it completes the circuit, indicating that the carriage has reached its travel limit. The controller must read this event and stop the motors immediately to prevent mechanical damage.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the active switch status.
2. **Emergency Overrun Timer**: Sound a continuous alarm if the limit switch remains closed for more than 5 seconds.
3. **RSSI Signal meter**: Check signal strength (RSSI) (Project 04) and flash an LED if the connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Switch states are inverted | Pull-down resistor missing | Confirm that a 10 kΩ pull-down resistor is installed on the switch pin |
| Switch bounces, sending multiple packets | Debounce delay too short | Increase the debounce delay in the code to 100 ms |
| Safety controller does not respond | Port mismatch | Ensure the controller is listening on port 9000 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [24 - ESP32 Wireless button trigger](24-esp32-wireless-button-trigger.md)
- [21 - ESP32 UDP Control Switch](21-esp32-udp-control-switch.md)
- [167 - ESP32 Current Overload Contactor Breaker](../expert/167-esp32-current-overload-contactor-breaker.md)
