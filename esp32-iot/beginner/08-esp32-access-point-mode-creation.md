# 08 - ESP32 Access Point Mode Creation

Configure the ESP32 in Access Point (AP) mode to host a local open wireless network, log the host configuration to the Serial Monitor, and track the number of connected clients in real time.

## Goal
Learn how to configure the ESP32 as a wireless Access Point (AP), customize local host parameters, and track connected clients.

## What You Will Build
An ESP32 DevKitC hosts an open local WiFi network named `ESP32-Open-AP` (no password). It configures the gateway at `192.168.4.1` and logs the network details. It monitors the network in the loop, printing the count of connected client devices (stations) whenever it changes.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in RF antenna to transmit the beacon frames.

## Code
```cpp
// Access Point mode creation (ESP32 as AP host)
#include <WiFi.h>

// Configuration parameters for the Access Point
const char* ap_ssid = "ESP32-Open-AP";

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Access Point (AP) Controller");
  Serial.println("==================================");
  
  // 1. Configure WiFi mode to Access Point (AP)
  WiFi.mode(WIFI_AP);
  
  // 2. Start the Access Point
  // softAP(ssid, password, channel, ssid_hidden, max_connection)
  // Password is set to NULL for an open (unsecured) network
  bool success = WiFi.softAP(ap_ssid, NULL);
  
  if (success) {
    Serial.println("Access Point hosted successfully!");
    
    // 3. Print AP configuration details
    Serial.print("SSID Name:  ");
    Serial.println(ap_ssid);
    
    Serial.print("AP IP Address: ");
    Serial.println(WiFi.softAPIP()); // Defaults to 192.168.4.1
    
    Serial.print("AP MAC Address: ");
    Serial.println(WiFi.softAPmacAddress());
  } else {
    Serial.println("Failed to initialize Access Point!");
    while(1) {}
  }
  
  Serial.println("==================================\n");
}

void loop() {
  static int lastStationCount = -1;
  
  // 4. Track connected stations (clients) in real time
  int currentStationCount = WiFi.softAPgetStationNum();
  
  if (currentStationCount != lastStationCount) {
    Serial.print("[AP Audit] Connected clients changed. Count: ");
    Serial.println(currentStationCount);
    lastStationCount = currentStationCount;
  }
  
  delay(1000); // Check client count every second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the AP IP and MAC print.
4. On hardware, search for WiFi networks on your phone or laptop. Verify that `ESP32-Open-AP` appears in the list.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Access Point (AP) Controller
==================================
Access Point hosted successfully!
SSID Name:  ESP32-Open-AP
AP IP Address: 192.168.4.1
AP MAC Address: 24:0A:C4:00:00:11
==================================

[AP Audit] Connected clients changed. Count: 0
[AP Audit] Connected clients changed. Count: 1
```

## Expected Canvas Behavior
* The Serial Monitor logs the hosted AP configuration (SSID, IP, MAC) and tracks connected clients.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_AP)` | Sets the ESP32 to act as a wireless Access Point (AP) host. |
| `WiFi.softAP(ap_ssid, NULL)` | Starts the AP broadcasting the specified SSID name with no password. |
| `WiFi.softAPIP()` | Returns the local IP address of the hosted AP (defaults to `192.168.4.1`). |
| `WiFi.softAPgetStationNum()` | Returns the number of client devices currently connected to the ESP32 AP. |

## Hardware & Safety Concept: Access Point (AP) Mode Hosting
Access Point (AP) mode (sometimes called softAP mode) allows the ESP32 to host its own wireless network. This is useful for standalone applications (like remote-controlling a robot in a field) or during initial product setup (allowing the user to connect and enter their home router credentials). Because open networks are unsecured, avoid transmitting sensitive data without encryption.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display the SSID, IP address, and the count of connected clients.
2. **Hidden network host**: Configure `WiFi.softAP` parameters to hide the SSID broadcast, requiring users to type the name manually.
3. **Client limit override**: Limit the maximum number of connected client devices to 3 to conserve ESP32 memory resources.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Failed to host Access Point | Mode mismatch | Ensure `WiFi.mode(WIFI_AP)` is called before starting `softAP` |
| IP address is not 192.168.4.1 | Custom IP configured | Use the default configuration or verify any custom subnet settings |
| The network is not visible to devices | Channel overlap | Change the channel parameter in `WiFi.softAP` to a cleaner channel (e.g. 1, 6, or 11) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [09 - ESP32 Access Point with Custom Security/Password](09-esp32-access-point-with-custom-security-password.md)
- [10 - ESP32 Access Point IP DHCP range logger](10-esp32-access-point-ip-dhcp-range-logger.md)
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
