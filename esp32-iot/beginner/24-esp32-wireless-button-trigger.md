# 24 - ESP32 Wireless Button Trigger

Configure the ESP32 to monitor a local push button, identify state changes (press events), and transmit a wireless UDP "TOGGLE" command packet to a remote receiver node on port 9000 to trigger a remote output.

## Goal
Learn how to implement state change detection (edge triggering), construct wireless trigger command packets, and transmit low-latency control codes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A push button is on GPIO 12. When the button is pressed (LOW to HIGH transition), the ESP32 transmits a UDP packet containing the string `TOGGLE` to the broadcast IP `255.255.255.255` on port 9000. This can be paired with a receiver node (like Project 19) to toggle an LED remotely.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Push Button | Pin 1 / Pin 2 | GPIO12 / 3V3 | Orange / Red | Wireless trigger button |
| Push Button | Pin 1 | GND via 10 kΩ | Black | Pull-down resistor |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the button with an external 10 kΩ pull-down resistor to GPIO 12.

## Code
```cpp
// Wireless button trigger (Transmit edge-triggered command)
#include <WiFi.h>
#include <WiFiUdp.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUTTON_PIN = 12;

// Target receiver details
// Using broadcast address to target any receiver listening on port 9000
IPAddress targetIP(255, 255, 255, 255);
const int targetPort = 9000;

WiFiUDP udp;

// Button state variables
bool lastButtonState = LOW;

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUTTON_PIN, INPUT); // External pull-down wired
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Wireless Button Trigger Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  udp.begin(8888); // Bind local port
  Serial.println("Ready. Press button to trigger remote.");
  Serial.println("==================================\n");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    // 1. Read current button state
    bool currentButtonState = (digitalRead(BUTTON_PIN) == HIGH);
    
    // 2. Identify State Change (LOW to HIGH edge detection)
    if (currentButtonState == HIGH && lastButtonState == LOW) {
      Serial.println("[Trigger] Button pressed! Transmitting TOGGLE packet...");
      
      // 3. Begin UDP packet
      udp.beginPacket(targetIP, targetPort);
      
      // 4. Write command string
      udp.print("TOGGLE");
      
      // 5. Transmit packet
      bool success = udp.endPacket();
      
      if (success) {
        Serial.println("[Trigger] Packet sent successfully.");
      } else {
        Serial.println("[Trigger] Packet send failed!");
      }
      
      delay(150); // Debounce delay
    }
    
    // Store active state for next comparison
    lastButtonState = currentButtonState;
  } else {
    Serial.println("WiFi offline!");
    delay(1000);
  }
  
  delay(10); // Yield to system
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Button** onto the canvas.
2. Wire Button to **GPIO12**.
3. Paste the code and click **Run**.
4. Click the button widget. Watch the Serial Monitor log the packet transmission.
5. In hardware, pair this with a receiver node (Project 19) to toggle an LED wirelessly.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Wireless Button Trigger Setup
==================================
WiFi Connected successfully.
Ready. Press button to trigger remote.
==================================

[Trigger] Button pressed! Transmitting TOGGLE packet...
[Trigger] Packet sent successfully.
```

## Expected Canvas Behavior
* Pressing the button widget triggers the UDP transmission log on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `currentButtonState == HIGH && lastButtonState == LOW` | Identifies the transition when the button is first pressed, ignoring hold states. |
| `udp.print("TOGGLE")` | Writes the command token into the packet buffer. |
| `udp.endPacket()` | Transmits the UDP packet to the broadcast IP on port 9000. |

## Hardware & Safety Concept: State-Change Detection (Edge Triggering) vs Level Triggering
In control systems, buttons must be edge-triggered (detecting the transition from LOW to HIGH). If the code uses level triggering (checking if the button is currently HIGH), holding the button down will cause the loop to run repeatedly, spamming the network with packets and locking up the receiver. Implementing state-change detection prevents network congestion.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a confirmation message when the button is pressed.
2. **Hold-to-Reset Command**: Send a different command (e.g. `RESET`) if the button is held down for more than 3 seconds.
3. **RSSI Signal meter**: Check signal strength (RSSI) (Project 04) and display it before sending packets.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Button presses are ignored | Pull-down resistor missing | Confirm that a 10 kΩ pull-down resistor is installed on the button pin |
| Packets are spammed | State-change logic error | Verify that `lastButtonState` is updated at the end of the loop |
| Receiver does not respond | Port mismatch | Ensure the receiver is listening on port 9000 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [18 - ESP32 UDP Packet Sender](18-esp32-udp-packet-sender.md)
- [19 - ESP32 UDP Packet Receiver](19-esp32-udp-packet-receiver.md)
- [25 - ESP32 Wireless buzzer receiver](25-esp32-wireless-buzzer-receiver.md)
