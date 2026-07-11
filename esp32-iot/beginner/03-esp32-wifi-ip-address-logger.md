# 03 - ESP32 WiFi IP Address Logger

Build a network details scanner on the ESP32 that extracts and logs the assigned DHCP Local IP address, Subnet Mask, Gateway IP, DNS Server IP, and the router AP MAC address (BSSID) to the Serial Monitor after connecting.

## Goal
Learn how to extract IP network configurations from the ESP32 WiFi stack, understand subnets and gateways, and log network details.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Once connected, it queries the local network stack to retrieve DHCP parameters (IP, subnet, gateway, DNS, and router BSSID) and logs them to the Serial Monitor every 10 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// WiFi IP Address Logger (DHCP IP details parser)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

void printNetworkDetails() {
  Serial.println("\n--- Connected Network Configuration ---");
  
  // 1. Get Local IP Address
  Serial.print("Local IP Address:  ");
  Serial.println(WiFi.localIP());
  
  // 2. Get Subnet Mask
  Serial.print("Subnet Mask:       ");
  Serial.println(WiFi.subnetMask());
  
  // 3. Get Default Gateway IP (Router address)
  Serial.print("Gateway IP:        ");
  Serial.println(WiFi.gatewayIP());
  
  // 4. Get DNS Server IP Address
  Serial.print("DNS Server IP:     ");
  Serial.println(WiFi.dnsIP());
  
  // 5. Get Router BSSID (Physical MAC Address of target router)
  Serial.print("Router MAC (BSSID): ");
  Serial.println(WiFi.BSSIDstr());
  
  // 6. Get ESP32 Local MAC Address
  Serial.print("ESP32 MAC Address: ");
  Serial.println(WiFi.macAddress());
  
  Serial.println("---------------------------------------\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing WiFi Network Scan...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  Serial.println("\nConnected successfully!");
  
  // Print network configuration details
  printNetworkDetails();
}

void loop() {
  // Periodically log details every 10 seconds
  static unsigned long lastLog = 0;
  if (millis() - lastLog >= 10000) {
    if (WiFi.status() == WL_CONNECTED) {
      printNetworkDetails();
    } else {
      Serial.println("Network offline. Reconnecting...");
    }
    lastLog = millis();
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the DHCP details log.

## Expected Output
Serial Monitor:
```
Initializing WiFi Network Scan...
....
Connected successfully!

--- Connected Network Configuration ---
Local IP Address:  10.10.0.3
Subnet Mask:       255.255.255.0
Gateway IP:        10.10.0.1
DNS Server IP:     10.10.0.1
Router MAC (BSSID): 02:00:00:00:00:00
ESP32 MAC Address: 24:0A:C4:00:00:10
---------------------------------------
```

## Expected Canvas Behavior
* The Serial Monitor logs the network configurations (IP, subnet, gateway, DNS, router BSSID, and local MAC) every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.subnetMask()` | Retrieves the subnet mask, which defines the network's IP address range. |
| `WiFi.gatewayIP()` | Retrieves the IP address of the gateway (router), which forwards local packets to the internet. |
| `WiFi.BSSIDstr()` | Returns the physical MAC address of the connected access point as a string. |

## Hardware & Safety Concept: DHCP vs Static IP configurations
When the ESP32 connects in Station mode, it defaults to using DHCP (Dynamic Host Configuration Protocol). The local router acts as a DHCP server, assigning the ESP32 a local IP address, subnet mask, gateway, and DNS servers. This prevents IP address conflicts on the network. For web servers that must remain at the same address, a static IP can be configured (Project 11).

## Try This! (Challenges)
1. **OLED Details Dashboard**: Add an OLED screen (Project 60) and display the IP address and MAC address on the screen.
2. **Connection Signal strength meter**: Check signal strength (RSSI) (Project 04) and display it alongside the network details.
3. **Static IP failover**: Program the ESP32 to assign itself a static IP address if the DHCP server fails to respond.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| IP address reads 0.0.0.0 | DHCP allocation delay | Wait a few seconds for the router to assign the IP address |
| Gateway IP reads 0.0.0.0 | Router configuration issue | Check router settings, ensuring DHCP is active |
| BSSID shows empty characters | Driver stack error | Ensure `WiFi.status()` is `WL_CONNECTED` before reading parameters |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [11 - ESP32 Static IP Address configuration](11-esp32-static-ip-address-configuration.md)
- [13 - ESP32 MAC Address resolver](13-esp32-mac-address-resolver.md)
