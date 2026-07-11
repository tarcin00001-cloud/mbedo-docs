# 174 - mDNS Local Domain Hostname setup (e.g. access via http://mbedo.local)

Build a local network domain name responder on the ESP32 using the Multicast DNS (mDNS) protocol, allowing users to access the ESP32's hosted web server using a friendly local address (`http://mbedo.local`) instead of its numeric IP address.

## Goal
Learn how to register Multicast DNS (mDNS) responders, advertise local TCP/UDP network services, set custom hostname strings, and host interactive local web nodes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It initializes the mDNS responder service with the hostname `mbedo`. An HTTP server is started on port 80, controlling an LED on GPIO 12. Computers and smartphones on the same WiFi network can open their web browsers and type `http://mbedo.local` to access the ESP32's control page, eliminating the need to search for the device's IP address.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Web-controlled light |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 220 Ω current-limiting resistor in series with the LED anode.

## Code
```cpp
// mDNS Local Domain Hostname setup (Multicast DNS responder + Web LED control)
#include <WiFi.h>
#include <ESPmDNS.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Friendly local hostname (accessible as http://mbedo.local)
const char* localHostname = "mbedo";

const int LED_PIN = 12;
bool ledState = false;

WebServer server(80);

// Serve main page
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Local Domain Console</title>\n";
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
  html += "  <h1>mDNS Local Domain Host</h1>\n";
  html += "  <div class=\"badge\">mDNS ACTIVE</div>\n";
  html += "  <p>You can access this device on your local network at:</p>\n";
  html += "  <a href=\"http://" + String(localHostname) + ".local/\" class=\"host-link\">http://" + String(localHostname) + ".local/</a>\n";
  
  // Interactive control form
  html += "  <form action=\"/toggle\" method=\"POST\">\n";
  html += "    <button class=\"btn " + String(ledState ? "active" : "") + "\" type=\"submit\">";
  html += ledState ? "TURN LED OFF" : "TURN LED ON";
  html += "    </button>\n";
  html += "  </form>\n";
  
  html += "  <p class=\"footer\">ESP32 IP: " + WiFi.localIP().toString() + " | Port 80</p>\n";
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
  
  Serial.println("\nESP32 mDNS local host node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 1. Initialize mDNS responder
  if (!MDNS.begin(localHostname)) {
    Serial.println("[mDNS Error] Failed to configure responder!");
  } else {
    Serial.printf("[mDNS] Responder active. Hostname: http://%s.local/\n", localHostname);
  }
  
  // 2. Start HTTP Server
  server.on("/", handleRoot);
  server.on("/toggle", HTTP_POST, handleToggle);
  server.begin();
  
  // 3. Add service to mDNS (makes it discoverable via UPnP/SSDP/Bonjour)
  MDNS.addService("http", "tcp", 80);
}

void loop() {
  server.handleClient();
  
  // No complex calls needed for mDNS in loop as the ESP32 driver runs in background
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to `http://mbedo.local/`.
5. Verify that the webpage loads and displays the current IP address.
6. Click the toggle button on the webpage. Verify that the simulated LED widget turns ON and OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[mDNS] Responder active. Hostname: http://mbedo.local/
```

## Expected Canvas Behavior
* Navigating to the mDNS URL opens the ESP32's control interface, and clicking the button toggles the LED widget state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `MDNS.begin(localHostname)` | Registers the device with the multicast DNS responder daemon. |
| `MDNS.addService("http", "tcp", 80)` | Advertises the HTTP service on port 80 to other devices on the network. |
| `server.sendHeader("Location", ...)` | Redirects the client back to the main page after performing an action. |

## Hardware & Safety Concept: Multicast DNS Protocol and Android Compatibility
* **Multicast DNS (mDNS)**: mDNS operates by sending UDP multicast packets on port 5353 to the local network IP `224.0.0.251`. Every device running an mDNS client (such as Apple's Bonjour or Avahi on Linux) listens to this address, allowing local name resolution without a central DNS server.
* **Android Compatibility**: Apple iOS and Windows 10/11 support mDNS natively. However, Google Android does not support local `.local` name resolution in standard browsers by default. To access local domains on Android, use a dedicated network exploration app or access the device directly via its IP address.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the mDNS address.
2. **SSDP Integration**: Add SSDP discovery (Project 175) to make the device appear in Windows "Network Places" automatically.
3. **Change Hostname online**: Create a web interface to change the local hostname and save the setting in NVS (Project 172).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| `mbedo.local` does not resolve | Host device doesn't support mDNS | Ensure mDNS is enabled on your host machine. Windows 10/11 supports it natively, but older Windows versions may require installing "Bonjour" services |
| Webpage loads slowly | mDNS query storm | Too many devices on the network broadcasting mDNS queries can cause packet loss. Try disabling unused network adapters on your computer |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - ESP32 DNS hostname configuration](../beginner/12-esp32-dns-hostname-configuration.md)
- [172 - Captive Portal Configuration Form](172-captive-portal-configuration-form.md)
- [175 - SSDP Device Discovery](175-ssdp-device-discovery.md) (Next project)
