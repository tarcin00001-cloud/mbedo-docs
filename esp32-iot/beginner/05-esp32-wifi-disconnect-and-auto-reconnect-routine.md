# 05 - ESP32 WiFi Disconnect and Polling Auto-reconnect Routine

Build a resilient connection manager on the ESP32 that uses loop-based status polling to detect WiFi disconnections and triggers auto-reconnect routines, using green and red LEDs for status indicators.

## Goal
Learn how to check WiFi status, handle network disconnections, design status polling reconnect loops, and manage connection indicators.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A Red LED is on GPIO 12, and a Green LED on GPIO 13. When connected, the Green LED is ON. If the router turns off, the ESP32 detects the disconnection via status polling, lights the Red LED, and attempts to reconnect every 5 seconds in the loop.

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
// WiFi Disconnect and Auto-reconnect routine (Loop-based Polling)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_RED = 12;   // Disconnected
const int LED_GREEN = 13; // Connected

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_RED, OUTPUT);
  pinMode(LED_GREEN, OUTPUT);
  
  // Start with Red LED on (offline)
  digitalWrite(LED_RED, HIGH);
  digitalWrite(LED_GREEN, LOW);
  
  Serial.println("\nResilient WiFi Connection Manager Starting...");
  
  // Configure STA mode and trigger connection
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
}

void loop() {
  // Monitor connection state and control LEDs
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(LED_GREEN, HIGH);
    digitalWrite(LED_RED, LOW);
    Serial.print("Assigned IP: ");
    Serial.println(WiFi.localIP());
    delay(5000); // Check status less frequently when connected
  } else {
    digitalWrite(LED_GREEN, LOW);
    digitalWrite(LED_RED, HIGH);
    Serial.println("Connection lost! Retrying WiFi connection...");
    WiFi.begin(ssid, password);
    delay(5000); // Wait 5 seconds before retrying
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
| `WiFi.status() == WL_CONNECTED` | Checks the Wi-Fi status against the connected enum. |
| `WiFi.begin(ssid, password)` | Begins the asynchronous connection to the access point. |
| `delay(5000)` | Wait interval before retrying connection or checking state. |

## Hardware & Safety Concept: Loop-Based Polling Network Programming
In simple designs, loops use blocking checks like `while(WiFi.status() != WL_CONNECTED)`. If the connection drops during operation, the controller locks up in this loop, halting motor control or sensor safety checks. Using loop-based status polling with non-blocking checks and `delay()` cycles, the ESP32 handles network status checks and reconnects gracefully.

## Try This! (Challenges)
1. **Buzzer disconnection alarm**: Sound a buzzer (GPIO 4) warning when the connection drops.
2. **Deep Sleep Fallback**: Enter deep sleep to conserve battery if the connection is down for more than 5 minutes.
3. **RSSI-triggered Reconnect**: Automatically trigger a reconnect if the signal strength (RSSI) drops below -85 dBm (Project 04).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reconnect triggers continuously | Credentials incorrect | Check that the SSID name and password characters match the router settings |
| System freezes during connection | Infinite loops in loop() | Keep loop steps non-blocking; do not use long delay times inside loop() |
| LED indicators don't update | LED polarity reversed | Verify the LED anode is connected to the GPIO and the cathode to ground |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [02 - ESP32 WiFi Connection Status Log](02-esp32-wifi-connection-status-log.md)
- [14 - ESP32 Wireless indicator LED](14-esp32-wireless-indicator-led.md)
