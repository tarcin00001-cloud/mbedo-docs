# 10 - ESP32 Access Point IP DHCP Range Logger

Configure the ESP32 in Access Point (AP) mode to host a secured local network with a custom IP address and subnet mask (overriding the default 192.168.4.1 configuration), and log client connection events.

## Goal
Learn how to configure custom IP subnets and gateway addresses for the ESP32 Access Point, and verify IP configurations.

## What You Will Build
An ESP32 DevKitC hosts a secured local WiFi network named `ESP32-Custom-Net` with the password `CustomPassword123`. Instead of the default `192.168.4.1` network, it configures a custom subnet (`10.10.10.1` with a subnet mask of `255.255.255.0`) and logs the hosted network details to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in RF antenna to host the network.

## Code
```cpp
// Access Point IP DHCP range logger (Custom Subnet AP)
#include <WiFi.h>

const char* ap_ssid = "ESP32-Custom-Net";
const char* ap_password = "CustomPassword123";

// Custom IP Address Configuration
IPAddress local_IP(10, 10, 10, 1);    // ESP32 AP Local IP
IPAddress gateway(10, 10, 10, 1);     // Gateway IP (usually matches local_IP)
IPAddress subnet(255, 255, 255, 0);   // Subnet Mask

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Custom AP Subnet Configurator");
  Serial.println("==================================");
  
  // 1. Configure WiFi mode to Access Point
  WiFi.mode(WIFI_AP);
  
  // 2. Set Custom IP Subnet Configuration
  // softAPConfig(local_ip, gateway, subnet)
  // Must be called before softAP()!
  bool configSuccess = WiFi.softAPConfig(local_IP, gateway, subnet);
  
  if (configSuccess) {
    Serial.println("Custom IP configuration applied.");
  } else {
    Serial.println("Custom IP configuration failed!");
  }
  
  // 3. Start the Access Point
  bool apSuccess = WiFi.softAP(ap_ssid, ap_password);
  
  if (apSuccess) {
    Serial.println("Access Point hosted successfully!");
    Serial.print("SSID:       "); Serial.println(ap_ssid);
    Serial.print("AP IP Address: "); Serial.println(WiFi.softAPIP()); // Returns 10.10.10.1
    Serial.print("Subnet Mask:   "); Serial.println(subnet);
    Serial.print("Gateway IP:    "); Serial.println(gateway);
  } else {
    Serial.println("Failed to start Access Point!");
    while(1) {}
  }
  Serial.println("==================================\n");
}

void loop() {
  static int lastClientCount = 0;
  int currentClientCount = WiFi.softAPgetStationNum();
  
  if (currentClientCount != lastClientCount) {
    Serial.print("[AP Info] Connected clients changed. Count: ");
    Serial.println(currentClientCount);
    lastClientCount = currentClientCount;
  }
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the custom AP IP (`10.10.10.1`) and subnet mask print.
4. On hardware, connect to the network. Verify that your device is assigned an IP in the `10.10.10.X` range (e.g. `10.10.10.2`).

## Expected Output
Serial Monitor:
```
==================================
ESP32 Custom AP Subnet Configurator
==================================
Custom IP configuration applied.
Access Point hosted successfully!
SSID:       ESP32-Custom-Net
AP IP Address: 10.10.10.1
Subnet Mask:   255.255.255.0
Gateway IP:    10.10.10.1
==================================

[AP Info] Connected clients changed. Count: 0
```

## Expected Canvas Behavior
* The Serial Monitor logs the custom AP configuration (SSID, IP, gateway, subnet) and tracks connected clients.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `IPAddress local_IP(...)` | Instantiates an IPAddress object with custom network coordinates. |
| `WiFi.softAPConfig(...)` | Configures the custom IP and subnet mask before hosting the network. |
| `WiFi.softAPIP()` | Verifies that the custom IP address has been applied. |

## Hardware & Safety Concept: Network Subnetting and Gateway Routing
By default, the ESP32 AP hosts a network in the `192.168.4.X` range. In industrial systems, this default configuration can cause conflicts with existing company subnets. Using `softAPConfig` allows defining custom subnet ranges (such as class A `10.X.X.X` or class B `172.16.X.X` private networks), integrating the ESP32 into existing network infrastructures.

## Try This! (Challenges)
1. **OLED Network Dashboard**: Add an OLED screen (Project 60) and display the custom IP address and subnet mask.
2. **Client IP Tracker**: Use ESP32 WiFi events (Project 05) to log the MAC address of clients when they connect.
3. **Class B Subnet configuration**: Configure the AP to use a Class B subnet (`172.16.0.1` with a subnet mask of `255.255.0.0`).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| AP IP defaults to 192.168.4.1 | Config called late | Ensure `WiFi.softAPConfig()` is called before `WiFi.softAP()` |
| Clients cannot connect | IP address mismatch | Verify that the gateway IP matches the AP local IP |
| System crashes on startup | Invalid IP coordinates | Verify that IP coordinates are set between 0 and 255 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 Access Point mode creation](08-esp32-access-point-mode-creation.md)
- [09 - ESP32 Access Point with Custom Security/Password](09-esp32-access-point-with-custom-security-password.md)
- [11 - ESP32 Static IP Address configuration](11-esp32-static-ip-address-configuration.md)
