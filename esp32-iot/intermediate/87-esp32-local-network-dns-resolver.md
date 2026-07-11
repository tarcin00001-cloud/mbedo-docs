# 87 - Local network DNS resolver

Configure the ESP32 to establish connections to local and public domain naming lookup engines, resolve hostnames (such as `google.com` and local `my-gateway.local` targets) to numerical IP addresses, and log the resolved coordinates.

## Goal
Learn how to use the DNS (Domain Name System) client resolver API, resolve public domain names, lookup local network hosts, and handle DNS resolution errors.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It uses the WiFi stack's built-in DNS client to resolve the public host `google.com` to its current IP address, and uses multicast DNS (mDNS) queries to lookup the local address of `my-gateway.local`, printing the resolved IP addresses to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Local network DNS resolver (Domain Name to IP Address resolver)
#include <WiFi.h>
#include <ESPmDNS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Hostnames to resolve
const char* targetPublicHost = "google.com";
const char* targetLocalHost = "my-gateway"; // Resolves to my-gateway.local

void resolveDomainName(const char* hostname) {
  IPAddress resolvedIP;
  
  Serial.print("[DNS] Querying IP for host: ");
  Serial.print(hostname);
  Serial.print("...");
  
  // 1. Resolve host name via DNS client resolver
  // Returns positive status if DNS lookup succeeds
  int result = WiFi.hostByName(hostname, resolvedIP);
  
  if (result == 1) {
    Serial.print(" Success! IP: ");
    Serial.println(resolvedIP);
  } else {
    Serial.println(" Failed! Host unreachable or DNS offline.");
  }
}

void resolveLocalMDNS(const char* serviceName) {
  Serial.print("[mDNS] Querying local host: ");
  Serial.print(serviceName);
  Serial.println(".local...");
  
  // 2. Query local network for mDNS service name
  // Returns the IP address of the target node if active
  IPAddress localIP = MDNS.queryHost(serviceName, 2000); // 2-second timeout
  
  if (localIP != IPAddress(0,0,0,0)) {
    Serial.print("[mDNS] Success! Local IP: ");
    Serial.println(localIP);
  } else {
    Serial.println("[mDNS] Failed! Node not found on local network.");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Local Network DNS Resolver");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 3. Initialize mDNS service client
  if (!MDNS.begin("esp32-resolver")) {
    Serial.println("[mDNS Error] Failed to start mDNS responder!");
  } else {
    Serial.println("[mDNS] Active. Hostname: esp32-resolver.local");
  }
  
  Serial.println("==================================\n");
  
  // Run public DNS resolution
  resolveDomainName(targetPublicHost);
  
  // Run local network mDNS query
  resolveLocalMDNS(targetLocalHost);
}

void loop() {
  // Static project. No execution loop tasks required.
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Watch the resolved IP coordinates log.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Local Network DNS Resolver
==================================
WiFi Connected.
[mDNS] Active. Hostname: esp32-resolver.local
==================================

[DNS] Querying IP for host: google.com... Success! IP: 142.250.190.46
[mDNS] Querying local host: my-gateway.local... Failed! Node not found on local network.
```

## Expected Canvas Behavior
* The Serial Monitor logs public DNS coordinates on boot.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.hostByName(...)` | Standard API to query configured DNS servers for the IP coordinates of a host name. |
| `MDNS.begin(...)` | Mounts the local Multicast DNS listener to allow resolve commands. |
| `MDNS.queryHost(...)` | Transmits multicast IP requests to local subnets to resolve hostnames. |

## Hardware & Safety Concept: DNS Cache and Timeouts in Web Clients
Opening TCP sockets requires first resolving hostnames. Because DNS queries traverse gateways and servers, they can block execution. Setting a **timeout limit** (e.g. 2000 ms in mDNS queries) ensures the system aborts the lookup and continues if the DNS server is offline.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the resolved IP coordinates.
2. **Periodic Diagnostic**: Program the ESP32 to query and ping the gateway IP every 5 minutes to verify connection health.
3. **Buzzer warning tone**: Sound a warning beep (GPIO 15) if DNS resolution fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Public DNS query fails | DNS server address missing | Verify that the WiFi gateway has assigned valid DNS server addresses via DHCP |
| mDNS query fails | Multicast blocked | Confirm that your router allows multicast transmission between devices |
| Resolution freezes | Infinite blocking call | Verify that query timeouts are configured |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - ESP32 DNS Hostname configuration](../beginner/12-esp32-dns-hostname-configuration.md)
- [16 - ESP32 TCP Client connection test](../beginner/16-esp32-tcp-client-connection-test.md)
- [31 - ESP32 HTTP GET Request](../beginner/31-esp32-http-get-request.md)
