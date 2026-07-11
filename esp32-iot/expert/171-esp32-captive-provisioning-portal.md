# 171 - ESP32 Captive Provisioning Portal

Build an offline Captive Portal on the ESP32 that runs a soft Access Point (AP) alongside a wildcard DNS server redirecting all domain requests to a custom HTTP landing configuration page, serving as the foundation for password-free device commissioning.

## Goal
Learn how to configure soft Access Point (AP) mode, implement wildcard DNS hijacking servers, intercept standard operating system connectivity checks, and design responsive landing interfaces.

## What You Will Build
An ESP32 DevKitC acts as a standalone wireless router, broadcasting a soft Access Point named `ESP32-Captive-Portal`. A wildcard DNS server listens on port 53. When a smartphone or computer connects, the operating system sends background connectivity checks (e.g. to Google or Apple). The DNS server intercepts these requests and responds with the ESP32's local gateway IP (`192.168.4.1`), triggering the system's captive portal browser window to display a styled welcome portal.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

No external physical sensors or components are required for this project. The ESP32 is powered via its Micro-USB port, operating as an autonomous wireless server.

> **Wiring tip:** When deploying in standalone environments, power the ESP32 using a stable 5V USB power bank or external regulated power source.

## Code
```cpp
// ESP32 Captive Provisioning Portal (Wildcard DNS Server + Soft AP Host + Portal Redirects)
#include <WiFi.h>
#include <DNSServer.h>
#include <WebServer.h>

// Captive Portal AP Settings
const char* apSSID = "ESP32-Captive-Portal";
const char* apPassword = ""; // Open network for easy initial commissioning

// Local IP Address of the ESP32 Access Point
const IPAddress apIP(192, 168, 4, 1);
const IPAddress subnet(255, 255, 255, 0);

// DNS Server on standard Port 53
const byte DNS_PORT = 53;
DNSServer dnsServer;

// HTTP Server on standard Port 80
WebServer server(80);

// Helper function to check if the client requested the local portal IP address
bool isLocalIP(String host) {
  return host == "192.168.4.1" || host == "localhost";
}

// Redirect all hijacked DNS requests to the local landing page
void handleRedirect() {
  String host = server.hostHeader();
  if (!isLocalIP(host)) {
    Serial.printf("[DNS Hijack] Redirecting request for http://%s/ to portal gateway.\n", host.c_str());
    server.sendHeader("Location", "http://192.168.4.1/", true);
    server.send(302, "text/plain", ""); // Redirect code
  } else {
    // Serve portal main page if accessing local IP directly
    handlePortalRoot();
  }
}

// Serve the styled captive portal landing page
void handlePortalRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Device Setup Portal</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  .logo { font-size: 48px; margin-bottom: 20px; }\n";
  html += "  h1 { font-size: 24px; color: #38bdf8; margin-top: 0; }\n";
  html += "  p { color: #94a3b8; font-size: 15px; line-height: 1.6; margin-bottom: 30px; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; text-decoration: none; box-sizing: border-box; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; background-color: #1e293b; color: #38bdf8; border: 1px solid #334155; margin-bottom: 20px; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <div class=\"logo\">📶</div>\n";
  html += "  <h1>ESP32 Provisioning Portal</h1>\n";
  html += "  <div class=\"badge\">DEVICE INITIALIZED</div>\n";
  html += "  <p>Welcome to your new IoT device. To configure network settings and connect this module to your local WiFi, click the button below to open the setup form.</p>\n";
  html += "  <a href=\"/configure\" class=\"btn\">Configure Device WiFi</a>\n";
  html += "  <p class=\"footer\">ESP32 Gateway IP: 192.168.4.1 | Soft AP Mode</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Serve a basic setup page
void handleConfigurePage() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Configuration Setup</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 20px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #10b981; border: none; border-radius: 6px; text-decoration: none; text-align: center; box-sizing: border-box; cursor: pointer; }\n";
  html += "  .btn-back { background-color: #334155; margin-top: 10px; }\n";
  html += "  p { color: #94a3b8; font-size: 14px; text-align: center; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>WiFi Network configuration</h1>\n";
  html += "  <p>To connect the device to your router, configure the next step of the provisioning sequence.</p>\n";
  html += "  <button class=\"btn\" onclick=\"alert('Ready to setup!')\">Save Credentials</button>\n";
  html += "  <a href=\"/\" class=\"btn btn-back\">Go Back</a>\n";
  html += "  <p style=\"font-size: 11px; margin-top: 20px; color: #64748b;\">Captive Portal Commissioning V1.0</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nConfiguring soft Access Point...");
  WiFi.mode(WIFI_AP);
  WiFi.softAPConfig(apIP, apIP, subnet);
  WiFi.softAP(apSSID, apPassword);
  
  delay(500);
  Serial.print("Access Point Broadcasing: ");
  Serial.println(apSSID);
  Serial.print("Gateway IP: ");
  Serial.println(WiFi.softAPIP());
  
  // Start DNS Server as a Wildcard "*" redirector
  // Intercepts all DNS requests and returns the local AP IP address
  dnsServer.start(DNS_PORT, "*", apIP);
  Serial.println("DNS server initialized (Wildcard mode).");
  
  // Web Server Routes
  server.on("/", handlePortalRoot);
  server.on("/configure", handleConfigurePage);
  
  // Wildcard handler intercepts all other HTTP requests and redirects them
  server.onNotFound(handleRedirect);
  
  server.begin();
  Serial.println("Captive web server started.");
}

void loop() {
  // Process incoming DNS redirects
  dnsServer.processNextRequest();
  
  // Handle HTTP client connections
  server.handleClient();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open a new tab in your web browser and attempt to navigate to a random domain (e.g. `http://apple.com` or `http://google.com`).
4. Verify that the browser gets hijacked, redirecting you instantly to the local landing portal page on `http://192.168.4.1/`.
5. Click "Configure Device WiFi" and verify that the page transitions to the configuration settings layout.

## Expected Output
Serial Monitor:
```
Configuring soft Access Point...
Access Point Broadcasting: ESP32-Captive-Portal
Gateway IP: 192.168.4.1
DNS server initialized (Wildcard mode).
Captive web server started.
[DNS Hijack] Redirecting request for http://connectivitycheck.gstatic.com/ to portal gateway.
```

## Expected Canvas Behavior
* Running the code triggers the ESP32 Soft AP. Navigating to any web address in the browser loads the ESP32's offline captive portal page.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFi.mode(WIFI_AP)` | Sets the ESP32 into standalone Access Point broadcast transmitter mode. |
| `dnsServer.start(53, "*", apIP)` | Hijacks all DNS queries on port 53, forcing them to resolve to `192.168.4.1`. |
| `server.hostHeader()` | Resolves the domain string requested by the client browser. |
| `server.onNotFound(handleRedirect)` | Catches all requests for arbitrary domains and redirects them to the local portal root. |

## Hardware & Safety Concept: Wildcard DNS Hijacking and Security Risks
* **Wildcard DNS Hijacking**: Operating systems detect captive portals by testing known plain HTTP URLs (like `http://detectportal.firefox.com` or `http://connectivitycheck.gstatic.com`). By intercepting these DNS calls and returning the ESP32's IP address, the OS realizes it is trapped behind a gateway and opens the login browser window.
* **Security Risks**: Captive portals operate without encryption (over HTTP, not HTTPS) because it is impossible to secure a self-signed SSL certificate for arbitrary hijacked domains. Never enter sensitive passwords, banking credentials, or personal keys on an unencrypted captive portal setup page.

## Try This! (Challenges)
1. **SSID scan list**: Add a WiFi scanning routine (Project 06) and list all local SSIDs on the landing page.
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound a chime whenever a client device successfully connects to the portal.
3. **SPIFFS integration**: Save the landing page HTML structure inside SPIFFS files (Project 62) instead of storing them as inline C++ string variables.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Redirect does not trigger | Browser is using HTTPS | Captive portal redirects only work on unencrypted HTTP requests (`http://`). If you try to access `https://google.com`, the browser will block the hijack as a security violation (HSTS) |
| Device fails to connect to AP | IP config conflict | Ensure your smartphone is not using a static IP configuration from a previous network. It must use DHCP to get its IP from the ESP32 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 Access Point mode creation](../beginner/08-esp32-access-point-mode-creation.md)
- [62 - ESP32 SPIFFS flash memory data logger](../intermediate/62-esp32-spiffs-flash-memory-data-logger.md)
- [172 - Captive Portal Configuration Form](172-captive-portal-configuration-form.md) (Next project)
