# 176 - LLMNR Hostname configuration

Build a Link-Local Multicast Name Resolution (LLMNR) and NetBIOS name responder on the ESP32 using the NetBIOS library, allowing Windows clients to resolve the ESP32's hostname (e.g., `http://mbedo`) and ping it directly without relying on DNS or mDNS suffixes.

## Goal
Learn how to configure NetBIOS name server daemons, resolve Link-Local Multicast Name Resolution (LLMNR) queries, map hostnames on local subnets, and host interactive web applications.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It starts a NetBIOS Name Service (NBNS) on port 137, registering the name `mbedo`. An HTTP server runs on port 80, controlling an LED on GPIO 12. Host devices on the same subnet can open a command line and type `ping mbedo`, or type `http://mbedo/` in their web browser to access the ESP32.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Solder a 220 Ω resistor in series with the LED anode to limit current.

## Code
```cpp
// NetBIOS / LLMNR Hostname configuration (NetBIOS Name Service + Web LED control)
#include <WiFi.h>
#include <NetBIOS.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NetBIOS Hostname (resolvable as http://mbedo/ without suffixes)
const char* netbiosHostname = "mbedo";

const int LED_PIN = 12;
bool ledState = false;

WebServer server(80);

// Serve main page
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>NetBIOS Domain Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 420px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .host-link { display: block; font-family: monospace; font-size: 18px; color: #38bdf8; text-decoration: none; padding: 10px; border-radius: 6px; background-color: #0f172a; border: 1px solid #334155; margin: 20px 0; }\n";
  html += "  .host-link:hover { color: #0ea5e9; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.active { background-color: #ef4444; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>NetBIOS / LLMNR Domain</h1>\n";
  html += "  <div class=\"badge\">NETBIOS ACTIVE</div>\n";
  html += "  <p>You can access this device on Windows networks at:</p>\n";
  html += "  <a href=\"http://" + String(netbiosHostname) + "/\" class=\"host-link\">http://" + String(netbiosHostname) + "/</a>\n";
  
  // Interactive control form
  html += "  <form action=\"/toggle\" method=\"POST\">\n";
  html += "    <button class=\"btn " + String(ledState ? "active" : "") + "\" type=\"submit\">";
  html += ledState ? "TURN LED OFF" : "TURN LED ON";
  html += "    </button>\n";
  html += "  </form>\n";
  
  html += "  <p class=\"footer\">ESP32 IP: " + WiFi.localIP().toString() + " | NBNS Port 137</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Handle LED toggle POST request
void handleToggle() {
  ledState = !ledState;
  digitalWrite(LED_PIN, ledState ? HIGH : LOW);
  
  // Redirect back to root page
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 NetBIOS/LLMNR Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // 1. Initialize NetBIOS Name Service (NBNS)
  // Registers the NetBIOS name on port 137
  NBNS.begin(netbiosHostname);
  Serial.printf("[NetBIOS] Registered name: %s\n", netbiosHostname);
  
  // 2. Start HTTP Server
  server.on("/", handleRoot);
  server.on("/toggle", HTTP_POST, handleToggle);
  server.begin();
  Serial.println("HTTP Server active.");
}

void loop() {
  server.handleClient();
  
  // NetBIOS operates in the background via the core library, no loop process required
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to `http://mbedo/`.
5. Verify that the webpage loads and displays the current IP address.
6. Click the toggle button on the webpage. Verify that the simulated LED widget turns ON and OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[NetBIOS] Registered name: mbedo
HTTP Server active.
```

## Expected Canvas Behavior
* Navigating to the NetBIOS domain URL opens the ESP32's control interface, and clicking the button toggles the LED widget state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `NBNS.begin(netbiosHostname)` | Registers the device with the NetBIOS Name Service. |
| `server.send(200, ...)` | Sends the HTML landing page to the client. |
| `WiFi.localIP().toString()` | Resolves the IP address of the ESP32. |

## Hardware & Safety Concept: NetBIOS vs. mDNS and Local Subnets
* **NetBIOS Name Service**: NetBIOS is a legacy name resolution protocol used primarily by Windows networks, running over UDP port 137. It allows devices to resolve names within a single local subnet without a DNS server.
* **Local Subnets**: NetBIOS and LLMNR operate using broadcast packets. Since broadcast packets do not cross router boundaries, these protocols only work for devices on the same local subnet. To access the device across different subnets, configure a static DNS entry on your router.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the NetBIOS domain name.
2. **SSDP Integration**: Add SSDP discovery (Project 175) to make the device appear in Windows "Network Places" automatically.
3. **Change Hostname online**: Create a web interface to change the NetBIOS hostname dynamically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `mbedo` does not resolve | NetBIOS disabled on host | Ensure NetBIOS over TCP/IP is enabled in the IPv4 settings of your network adapter on Windows |
| Connection times out | Firewall blocking port 137 | Ensure your router or firewall is not blocking UDP port 137, which is required for NetBIOS name resolution |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - ESP32 DNS hostname configuration](../beginner/12-esp32-dns-hostname-configuration.md)
- [174 - ESP32 mDNS local domain hostname setup](174-mdns-local-domain-hostname-setup.md)
- [177 - HTTPS Server with Self-Signed Certificate](177-https-server-with-self-signed-certificate.md) (Next project)
