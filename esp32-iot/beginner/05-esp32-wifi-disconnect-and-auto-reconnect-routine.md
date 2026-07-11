# 05 - ESP32 WiFi Disconnect and Auto-reconnect Routine

Build a resilient connection manager on the ESP32 that uses event-driven callbacks to detect WiFi disconnections and triggers non-blocking auto-reconnect routines, using green and red LEDs for status indicators.

## Goal
Learn how to register WiFi event callbacks, handle network disconnections, design non-blocking reconnect loops, and manage connection indicators.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A Red LED is on GPIO 12, and a Green LED on GPIO 13. When connected, the Green LED is ON. If the router turns off, the ESP32 registers the disconnection event, lights the Red LED, and attempts to reconnect every 5 seconds in the background without blocking the main execution loop.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO12 via 330 Ω | Red | Disconnected status |
| Green LED | Anode (+) | GPIO13 via 330 Ω | Green | Connected status |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the LEDs with current-limiting 330 Ω resistors.

## Code
```cpp
// WiFi Disconnect and Auto-reconnect routine (Event-driven callbacks)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_RED = 12;   // Disconnected
const int LED_GREEN = 13; // Connected

// Non-blocking timer variables
unsigned long lastReconnectAttempt = 0;
const unsigned long RECONNECT_INTERVAL_MS = 5000; // Retry every 5 seconds

// WiFi Event Callback handler
// Automatically called by the ESP32 WiFi stack on network events
void WiFiEvent(WiFiEvent_t event) {
  switch (event) {
    case SYSTEM_EVENT_STA_START:
      Serial.println("[WiFi Event] Station Mode Started.");
      break;
      
    case SYSTEM_EVENT_STA_CONNECTED:
      Serial.println("[WiFi Event] Connected to access point.");
      break;
      
    case SYSTEM_EVENT_STA_GOT_IP:
      Serial.print("[WiFi Event] Assigned IP: ");
      Serial.println(WiFi.localIP());
      
      // Update LED indicators
      digitalWrite(LED_GREEN, HIGH);
      digitalWrite(LED_RED, LOW);
      break;
      
    case SYSTEM_EVENT_STA_DISCONNECTED:
      Serial.println("[WiFi Event] Connection lost! Triggering auto-reconnect...");
      
      // Update LED indicators
      digitalWrite(LED_GREEN, LOW);
      digitalWrite(LED_RED, HIGH);
      
      // Trigger background reconnect
      WiFi.begin(ssid, password);
      break;
      
    default:
      break;
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  
  // Start with Red LED on (offline)
  digitalWrite(LED_RED, HIGH);
  digitalWrite(LED_GREEN, LOW);
  
  Serial.println("\nResilient WiFi Connection Manager Starting...");
  
  // 1. Register the WiFi event callback function
  WiFi.onEvent(WiFiEvent);
  
  // 2. Configure STA mode and trigger connection
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
}

void loop() {
  // 3. Monitor connection state non-blockingly
  bool isConnected = (WiFi.status() == WL_CONNECTED);
  unsigned long now = millis();
  
  if (!isConnected) {
    // If disconnected, flash the Red LED every 500 ms
    static unsigned long lastFlash = 0;
    if (now - lastFlash >= 500) {
      digitalWrite(LED_RED, !digitalRead(LED_RED));
      lastFlash = now;
    }
    
    // Periodically re-trigger WiFi.begin() if the automatic stack reconnect stalls
    if (now - lastReconnectAttempt >= RECONNECT_INTERVAL_MS) {
      Serial.println("[Main Loop] Retrying WiFi connection...");
      WiFi.begin(ssid, password);
      lastReconnectAttempt = now;
    }
  }
  
  // 4. Main application code runs here without blocking!
  static unsigned long lastWork = 0;
  if (now - lastWork >= 1000) {
    Serial.print("[System Run] Active Time: ");
    Serial.print(now / 1000);
    Serial.print("s | Status: ");
    Serial.println(isConnected ? "ONLINE" : "OFFLINE");
    lastWork = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, and two **LEDs** onto the canvas.
2. Wire Red LED to **GPIO12** and Green LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Inspect the Serial Monitor. Watch the event logs.
5. In simulation, toggle the virtual WiFi network off. Watch the Green LED turn OFF, the Red LED flash, and the ESP32 trigger reconnect attempts.

## Expected Output
Serial Monitor:
```
Resilient WiFi Connection Manager Starting...
[WiFi Event] Station Mode Started.
[System Run] Active Time: 1s | Status: OFFLINE
[WiFi Event] Connected to access point.
[WiFi Event] Assigned IP: 10.10.0.3
[System Run] Active Time: 2s | Status: ONLINE
[System Run] Active Time: 3s | Status: ONLINE
[WiFi Event] Connection lost! Triggering auto-reconnect...
[System Run] Active Time: 4s | Status: OFFLINE
[Main Loop] Retrying WiFi connection...
```

## Expected Canvas Behavior
* At boot, the Red LED flashes.
* Once connected, the Red LED turns OFF and the Green LED glows solid green.
* Toggling the simulated network off turns the Green LED OFF and causes the Red LED to flash.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.onEvent(WiFiEvent)` | Registers the callback handler to receive system network events. |
| `SYSTEM_EVENT_STA_GOT_IP` | Triggers when the DHCP server assigns an IP address. |
| `SYSTEM_EVENT_STA_DISCONNECTED` | Triggers when the link drops, starting the auto-reconnect sequence. |
| `now - lastReconnectAttempt` | Runs the reconnect attempts in the background, keeping the loop active. |

## Hardware & Safety Concept: Event-Driven Network Programming
In simple designs, loops use blocking checks like `while(WiFi.status() != WL_CONNECTED)`. If the connection drops during operation, the controller locks up in this loop, halting motor control or sensor safety checks. Using **Event-Driven WiFi Programming** and non-blocking timers, the ESP32 handles network reconnections in the background, letting the main control loop continue running.

## Try This! (Challenges)
1. **Buzzer disconnection alarm**: Sound a buzzer (GPIO 4) warning when the connection drops.
2. **Deep Sleep Fallback**: Enter deep sleep to conserve battery if the connection is down for more than 5 minutes.
3. **RSSI-triggered Reconnect**: Automatically trigger a reconnect if the signal strength (RSSI) drops below -85 dBm (Project 04).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reconnect triggers continuously | Credentials incorrect | Check that the SSID name and password characters match the router settings |
| System freezes during reconnection | Blocking calls in callback | Keep callback handlers simple; do not use delays or slow serial operations inside the event handler |
| LED indicators don't update | LED polarity reversed | Verify the LED anode is connected to the GPIO and the cathode to ground |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [02 - ESP32 WiFi Connection Status Log](02-esp32-wifi-connection-status-log.md)
- [14 - ESP32 Wireless indicator LED](14-esp32-wireless-indicator-led.md)
