# 01 - ESP32 WiFi Station Connection

Configure the ESP32 in Station (STA) mode to connect to a local WiFi network, using the serial monitor to log the handshake status and blinking the onboard LED during the connection process.

## Goal
Learn how to initialize the ESP32 WiFi library, set up network credentials, change WiFi modes to Station (STA), and verify connection handshakes.

## What You Will Build
An ESP32 DevKitC connects to a simulated or real local WiFi router (Access Point). The onboard LED (GPIO 2) blinks rapidly while searching for the network and turns solid HIGH once connected. The IP address and connection details are printed to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required for this introductory lab. The ESP32 uses its built-in PCB antenna and the onboard Blue LED (GPIO 2) for status.

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Onboard LED | Anode (+) | GPIO2 | Internal | Status LED |

> **Wiring tip:** The blue status LED on the ESP32 DevKitC is internally connected to GPIO 2. No external resistor or LED is needed.

## Code
```cpp
// ESP32 WiFi Station Connection (Join local SSID)
#include <WiFi.h>

// Enter your WiFi router credentials
const char* ssid = "Wokwi-GUEST"; // Default simulated network in Wokwi/MbedO
const char* password = "";        // Guest networks often have empty passwords

const int STATUS_LED = 2; // Onboard LED

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 WiFi Station Connection");
  Serial.println("==================================");
  
  // 1. Configure WiFi mode to Station (client)
  WiFi.mode(WIFI_STA); 
  
  // 2. Start connection sequence
  Serial.print("Connecting to SSID: ");
  Serial.println(ssid);
  WiFi.begin(ssid, password);
}

void loop() {
  // 3. Monitor connection status inside loop
  if (WiFi.status() == WL_CONNECTED) {
    digitalWrite(STATUS_LED, HIGH);
    Serial.println("\nConnection Successful!");
    Serial.print("Assigned IP: ");
    Serial.println(WiFi.localIP());
    delay(5000); // Check status less frequently once connected
  } else {
    // Blink LED while connecting
    digitalWrite(STATUS_LED, HIGH);
    delay(250);
    digitalWrite(STATUS_LED, LOW);
    delay(250);
    Serial.print(".");
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the connection logs print.
4. Verify that the onboard LED widget (GPIO 2) blinks during the handshake and turns solid blue once connected.

## Expected Output
Serial Monitor:
```
==================================
ESP32 WiFi Station Connection
==================================
Connecting to SSID: Wokwi-GUEST
......

Connection Successful!
Assigned IP: 10.10.0.3
```

## Expected Canvas Behavior
* On boot, the onboard Blue LED (GPIO 2) flashes.
* Once the virtual network handshake completes, the LED glows solid blue.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_STA)` | Configures the ESP32 as a client station (STA), rather than an Access Point (AP). |
| `WiFi.begin(ssid, ...)` | Triggers the background connection sequence to the specified access point. |
| `WiFi.status() != WL_CONNECTED` | Checks the wireless controller status; returns `WL_CONNECTED` once the handshake is complete. |
| `WiFi.localIP()` | Retrieves the IP address assigned to the ESP32 by the network DHCP server. |

## Hardware & Safety Concept: WiFi Modes and Antenna Placement
The ESP32 contains an onboard 2.4 GHz RF radio chip and a printed PCB antenna. When designing wireless products:
1. **Antenna clearance**: Keep metal components, copper traces, and batteries away from the PCB antenna. Placing metal near the antenna absorbs the RF energy, reducing the connection range.
2. **WiFi Modes**: Use `WIFI_STA` for client mode (connecting to a router) and `WIFI_AP` for hosting a network. Explicitly setting the mode on startup prevents the radio from consuming unnecessary power by hosting a default network.

## Try This! (Challenges)
1. **Dynamic Timeout**: Modify the code to retry connecting for 60 seconds before giving up and entering deep sleep to save power.
2. **Network selector**: Store two alternative WiFi credentials, trying the second one if the first fails.
3. **Signal buzzer beep**: Connect an external buzzer (GPIO 15) and beep three times when the connection is established.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Endless dots in Serial Monitor | Credentials incorrect | Check that the SSID name and password characters match the router settings |
| ESP32 reboots continuously | Low USB power | WiFi transmissions draw up to 250 mA. Connect the ESP32 directly to a USB port capable of supplying 500 mA |
| WiFi status is always disconnected | Router blocking client | Verify that the router's MAC filter or DHCP settings allow new client connections |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [02 - ESP32 WiFi Connection Status Log](02-esp32-wifi-connection-status-log.md)
- [05 - ESP32 WiFi Disconnect and Polling Auto-reconnect routine](05-esp32-wifi-disconnect-and-auto-reconnect-routine.md)
- [14 - ESP32 Wireless indicator LED](14-esp32-wireless-indicator-led.md)
