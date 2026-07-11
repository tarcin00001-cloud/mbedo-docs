# 11 - ESP32 Static IP Address Configuration

Configure the ESP32 in Station (STA) mode to override DHCP configuration and assign itself a static IP address, gateway, subnet mask, and DNS server, logging network details to the Serial Monitor.

## Goal
Learn how to override DHCP configurations on the ESP32 WiFi client, set static IP parameters, and integrate with local subnets.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Instead of waiting for DHCP to assign an IP, the ESP32 configures its own static IP address (`10.10.0.184` with a gateway of `10.10.0.1` and subnet mask of `255.255.255.0`) and logs the verified connection details.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Static IP Address configuration (Override DHCP IP)
#include <WiFi.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configure Static IP network coordinates
// Match the subnet of your local router (usually 192.168.1.X or 10.10.0.X)
IPAddress local_IP(10, 10, 0, 184);   // Target static IP for the ESP32
IPAddress gateway(10, 10, 0, 1);      // Router IP address
IPAddress subnet(255, 255, 255, 0);   // Subnet mask
IPAddress primaryDNS(10, 10, 0, 1);   // DNS Server IP (usually matches gateway)
IPAddress secondaryDNS(8, 8, 8, 8);   // Backup Google DNS

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Static IP Configurator");
  Serial.println("==================================");
  
  // 1. Configure WiFi mode to Station
  WiFi.mode(WIFI_STA);
  
  // 2. Set Static IP Configuration
  // config(local_ip, gateway, subnet, dns1, dns2)
  // Must be called before WiFi.begin()!
  if (WiFi.config(local_IP, gateway, subnet, primaryDNS, secondaryDNS)) {
    Serial.println("Static IP configuration applied successfully.");
  } else {
    Serial.println("Failed to apply Static IP configuration!");
  }
  
  // 3. Connect to WiFi network
  Serial.print("Connecting to SSID: "); Serial.println(ssid);
  WiFi.begin(ssid, password);
  
  // Wait for connection
  int attempts = 0;
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\nConnection Successful!");
    
    // 4. Verify assigned network details
    Serial.print("Verified IP Address: "); Serial.println(WiFi.localIP()); // Returns 10.10.0.184
    Serial.print("Subnet Mask:         "); Serial.println(WiFi.subnetMask());
    Serial.print("Gateway IP:          "); Serial.println(WiFi.gatewayIP());
    Serial.print("DNS IP:              "); Serial.println(WiFi.dnsIP());
  } else {
    Serial.println("\nConnection Timeout! Check credentials and subnet IP coordinates.");
  }
  Serial.println("==================================\n");
}

void loop() {
  // Idle
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the assigned IP address matches `10.10.0.184`.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Static IP Configurator
==================================
Static IP configuration applied successfully.
Connecting to SSID: Wokwi-GUEST
....
Connection Successful!
Verified IP Address: 10.10.0.184
Subnet Mask:         255.255.255.0
Gateway IP:          10.10.0.1
DNS IP:              10.10.0.1
==================================
```

## Expected Canvas Behavior
* The Serial Monitor logs the static IP configurations (IP, gateway, subnet, and DNS) once connected.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.config(...)` | Configures the static IP and network parameters before establishing the connection. |
| `WiFi.localIP()` | Verifies that the configured static IP address has been applied. |

## Hardware & Safety Concept: Why and When to Use Static IP Addresses
When acting as a server (such as hosting a smart home dashboard or API node), the ESP32 must remain accessible at the same address. If using DHCP, the router can assign a new IP address on reboot, disconnecting client devices. Assigning a **Static IP** ensures the ESP32's address remains constant. To prevent address conflicts, select a static IP address outside the router's dynamic DHCP pool range.

## Try This! (Challenges)
1. **Interactive OLED Monitor**: Add an OLED screen (Project 60) and display the verified static IP address.
2. **Fallback to DHCP**: Program the ESP32 to retry using DHCP if the static IP configuration fails to establish a connection.
3. **Address conflict detector**: Ping the target IP address before assigning it to ensure it is not already in use by another device.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with static IP, but works with DHCP | Subnet IP mismatch | Verify that the static IP matches the router subnet range (e.g. check if gateway matches router IP) |
| Static IP config ignored | Config called late | Ensure `WiFi.config()` is called before `WiFi.begin()` |
| DNS resolution fails | DNS IP incorrect | Set the primary DNS address to the router IP or a public server (e.g. 8.8.8.8) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [01 - ESP32 WiFi Station Connection](01-esp32-wifi-station-connection.md)
- [03 - ESP32 WiFi IP Address Logger](03-esp32-wifi-ip-address-logger.md)
- [10 - ESP32 Access Point IP DHCP range logger](10-esp32-access-point-ip-dhcp-range-logger.md)
