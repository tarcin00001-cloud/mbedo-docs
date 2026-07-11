# 12 - ESP32 DNS Hostname Configuration

Configure the ESP32 to run a Multicast DNS (mDNS) responder service, allowing local network devices to access the ESP32 using a custom hostname URL (e.g. `http://esp32-controller.local`) rather than its numeric IP address.

## Goal
Learn how to initialize the mDNS library, configure local hostnames, and resolve network nodes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It initializes the mDNS responder service with the hostname `esp32-controller`. Once initialized, it logs the local access URL to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// DNS Hostname configuration (mDNS responder service)
#include <WiFi.h>
#include <ESPmDNS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Target local hostname (e.g. http://esp32-controller.local)
const char* hostname = "esp32-controller";

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 mDNS Hostname Configurator");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  // Wait for connection
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi connected successfully!");
  Serial.print("DHCP IP Address: "); Serial.println(WiFi.localIP());
  
  // 1. Initialize mDNS responder service
  Serial.print("Initializing mDNS responder for hostname: ");
  Serial.println(hostname);
  
  if (MDNS.begin(hostname)) {
    Serial.println("mDNS responder service started successfully.");
    
    // 2. Log local access URL
    Serial.print("Local Network Access Address: http://");
    Serial.print(hostname);
    Serial.println(".local/");
    
    // 3. Add service (optional: advertises HTTP service on port 80)
    MDNS.addService("http", "tcp", 80);
  } else {
    Serial.println("Error setting up mDNS responder!");
  }
  Serial.println("==================================\n");
}

void loop() {
  // In ESP32 Arduino Core, the mDNS responder runs in the background.
  // No MDNS.update() call is required.
  delay(5000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the access URL prints.
4. On hardware, connect a laptop to the same network and ping `esp32-controller.local` to verify resolution.

## Expected Output
Serial Monitor:
```
==================================
ESP32 mDNS Hostname Configurator
==================================
WiFi connected successfully!
DHCP IP Address: 10.10.0.3
Initializing mDNS responder for hostname: esp32-controller
mDNS responder service started successfully.
Local Network Access Address: http://esp32-controller.local/
==================================
```

## Expected Canvas Behavior
* The Serial Monitor logs the verified mDNS hostname configuration and access address.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `MDNS.begin(hostname)` | Initializes the mDNS responder service with the specified hostname string. |
| `MDNS.addService(...)` | Advertises the HTTP TCP port 80 service, helping discover the device on the network. |

## Hardware & Safety Concept: Multicast DNS (mDNS) Routing
Numeric IP addresses (like `192.168.1.184`) are difficult to remember and change dynamically when using DHCP. **Multicast DNS (mDNS)** resolves hostnames locally using the `.local` top-level domain. When a device requests `esp32-controller.local`, the ESP32 intercepts the multicast request and returns its IP address. This removes the need for a local DNS server.

## Try This! (Challenges)
1. **OLED URL Display**: Add an OLED screen (Project 60) and display the mDNS address URL.
2. **Device Finder Service**: Add mDNS TXT records to advertise custom device metadata (e.g. firmware version).
3. **Dynamic Hostname ID**: Append the last two bytes of the ESP32 MAC address to the hostname to ensure it remains unique on the network.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Hostname does not resolve | Router blocking multicast | Verify that the router's settings allow multicast transmissions |
| mDNS initialization fails | Hostname empty or too long | Keep the hostname under 63 characters and avoid special characters or spaces |
| Windows PC cannot resolve `.local` | mDNS service disabled | Install Apple Bonjour Print Services or enable mDNS on Windows to resolve local hosts |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [03 - ESP32 WiFi IP Address Logger](03-esp32-wifi-ip-address-logger.md)
- [11 - ESP32 Static IP Address configuration](11-esp32-static-ip-address-configuration.md)
- [51 - ESP32 Single Page HTTP Web Server](../../intermediate/51-esp32-single-page-http-web-server.md)
