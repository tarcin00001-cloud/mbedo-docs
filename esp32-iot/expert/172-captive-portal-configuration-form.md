# 172 - Captive Portal Configuration Form (Saves WiFi credentials to EEPROM/NVS)

Build a complete WiFi provisioning gateway on the ESP32 that runs a Captive Portal, serves an HTML configuration form allowing users to select or type their WiFi SSID and Password, saves the credentials to Non-Volatile Storage (NVS) using the Preferences library, and reboots to establish connection in Station mode.

## Goal
Learn how to use the ESP32 NVS (Non-Volatile Storage) flash partition using the Preferences library, parse HTTP POST form variables, manage boot configuration states, and implement automatic connection fallback loops.

## What You Will Build
An ESP32 DevKitC boots up. It checks NVS memory for saved WiFi credentials. If none are found, it starts a Soft AP named `ESP32-Setup` and runs a DNS wildcard server. When a smartphone connects, it displays a portal webpage with an input form. The user submits their SSID and Password. The ESP32 saves them to NVS and reboots. On the next boot, it loads the credentials and connects to the router, switching a status LED on GPIO 12 ON.

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

> **Wiring tip:** Use a 220 Ω current-limiting resistor in series with the LED anode.

## Code
```cpp
// Captive Portal Configuration Form (NVS storage via Preferences + Wildcard DNS + Soft AP Fallbacks)
#include <WiFi.h>
#include <DNSServer.h>
#include <WebServer.h>
#include <Preferences.h>

const int STATUS_LED = 12;

// Soft AP Config (Used if no router credentials are saved)
const char* apSSID = "ESP32-Setup-Portal";
const IPAddress apIP(192, 168, 4, 1);
const IPAddress subnet(255, 255, 255, 0);

DNSServer dnsServer;
WebServer server(80);
Preferences preferences;

// Global settings
String routerSSID = "";
String routerPass = "";
bool isConfiguredMode = false;

// Read config parameters from NVS
void loadCredentials() {
  preferences.begin("wifi-creds", true); // Open namespace in read-only mode
  routerSSID = preferences.getString("ssid", "");
  routerPass = preferences.getString("pass", "");
  preferences.end();
  
  if (routerSSID.length() > 0) {
    isConfiguredMode = true;
    Serial.printf("[NVS] Loaded saved credentials: SSID: %s\n", routerSSID.c_str());
  } else {
    isConfiguredMode = false;
    Serial.println("[NVS] No saved credentials found. Defaulting to setup mode.");
  }
}

// Intercept wildcard DNS requests and redirect to gateway IP
void handleRedirect() {
  String host = server.hostHeader();
  if (host != "192.168.4.1" && host != "localhost") {
    server.sendHeader("Location", "http://192.168.4.1/", true);
    server.send(302, "text/plain", ""); // Redirect to portal root
  } else {
    handlePortalRoot();
  }
}

// Serve landing configuration page
void handlePortalRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>WiFi Provisioning Portal</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; margin-bottom: 25px; }\n";
  html += "  .form-group { margin-bottom: 20px; }\n";
  html += "  label { display: block; font-size: 13px; color: #94a3b8; font-weight: bold; margin-bottom: 8px; text-transform: uppercase; }\n";
  html += "  input[type=text], input[type=password] { width: 100%; padding: 12px; border-radius: 6px; border: 1px solid #334155; background-color: #0f172a; color: #f8fafc; font-size: 15px; box-sizing: border-box; outline: none; }\n";
  html += "  input[type=text]:focus, input[type=password]:focus { border-color: #38bdf8; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #10b981; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-top: 10px; }\n";
  html += "  .btn:hover { background-color: #059669; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>WiFi Commissioning</h1>\n";
  html += "  <form action=\"/save\" method=\"POST\">\n";
  html += "    <div class=\"form-group\"><label>WiFi Network Name (SSID)</label><input type=\"text\" name=\"ssid\" placeholder=\"SSID Name\" required></div>\n";
  html += "    <div class=\"form-group\"><label>WiFi Password</label><input type=\"password\" name=\"password\" placeholder=\"Password\" required></div>\n";
  html += "    <button class=\"btn\" type=\"submit\">Save & Connect</button>\n";
  html += "  </form>\n";
  html += "  <p class=\"footer\">ESP32 Gateway: 192.168.4.1</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Handle credential save API POST request
void handleSaveCredentials() {
  if (server.hasArg("ssid") && server.hasArg("password")) {
    String newSsid = server.arg("ssid");
    String newPass = server.arg("password");
    
    // Save to NVS partition
    preferences.begin("wifi-creds", false); // Open namespace in read-write mode
    preferences.putString("ssid", newSsid);
    preferences.putString("pass", newPass);
    preferences.end();
    
    Serial.println("[NVS] Credentials written successfully.");
    
    String html = "<html><head><meta http-equiv='refresh' content='4;url=/'></head>";
    html += "<body style='background-color:#0f172a;color:#f8fafc;font-family:sans-serif;text-align:center;padding:50px;'>";
    html += "<h2>Configuration saved!</h2><p>Rebooting device to connect to network. Please wait...</p></body></html>";
    server.send(200, "text/html", html);
    
    delay(2000);
    ESP.restart(); // Reboot ESP32
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Attempt to connect to saved router credentials
bool connectToWiFi() {
  WiFi.mode(WIFI_STA);
  WiFi.begin(routerSSID.c_str(), routerPass.c_str());
  
  Serial.printf("[STA] Connecting to: %s ", routerSSID.c_str());
  int attempts = 0;
  // Try connecting for 15 seconds
  while (WiFi.status() != WL_CONNECTED && attempts < 30) {
    delay(500);
    Serial.print(".");
    attempts++;
  }
  
  if (WiFi.status() == WL_CONNECTED) {
    Serial.println("\n[STA] WiFi Connected successfully!");
    Serial.print("[STA] Local IP Address: ");
    Serial.println(WiFi.localIP());
    digitalWrite(STATUS_LED, HIGH); // LED ON indicates successful router link
    return true;
  } else {
    Serial.println("\n[STA Error] Connection timed out! Reverting to soft AP mode.");
    digitalWrite(STATUS_LED, LOW);
    return false;
  }
}

// Start setup AP and hijack DNS
void startSetupPortal() {
  WiFi.mode(WIFI_AP);
  WiFi.softAPConfig(apIP, apIP, subnet);
  WiFi.softAP(apSSID);
  
  dnsServer.start(53, "*", apIP);
  
  server.on("/", handlePortalRoot);
  server.on("/save", HTTP_POST, handleSaveCredentials);
  server.onNotFound(handleRedirect);
  
  server.begin();
  Serial.println("[Portal] Captive server active on http://192.168.4.1/");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  // 1. Read config credentials
  loadCredentials();
  
  // 2. State selection
  if (isConfiguredMode) {
    // Attempt to connect
    if (!connectToWiFi()) {
      // Revert if connection failed
      startSetupPortal();
    }
  } else {
    startSetupPortal();
  }
}

void loop() {
  // If in AP setup mode, process redirect queries
  if (WiFi.getMode() == WIFI_AP || WiFi.getMode() == WIFI_AP_STA) {
    dnsServer.processNextRequest();
    server.handleClient();
  } else {
    // Standard application loops if connected
    static unsigned long lastBlink = 0;
    if (millis() - lastBlink > 2000) {
      lastBlink = millis();
      Serial.println("[System] Active & Connected. Router IP: " + WiFi.localIP().toString());
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. In normal state, since NVS starts empty, verify that the console shows `No saved credentials found.` and the portal starts up.
5. Open your web browser and navigate to `http://192.168.4.1/`.
6. Fill in the network SSID (e.g. `Wokwi-GUEST`) and password. Click "Save & Connect".
7. Verify that the console prints `Credentials written successfully` and the ESP32 reboots.
8. On the reboot run, verify that the ESP32 loads the saved credentials, connects successfully, and the status LED turns ON.

## Expected Output
Serial Monitor (First Boot):
```
No saved credentials found. Defaulting to setup mode.
[Portal] Captive server active on http://192.168.4.1/
[NVS] Credentials written successfully.
```

Serial Monitor (Second Boot after restart):
```
[NVS] Loaded saved credentials: SSID: Wokwi-GUEST
[STA] Connecting to: Wokwi-GUEST ...........
[STA] WiFi Connected successfully!
[STA] Local IP Address: 10.10.0.3
```

## Expected Canvas Behavior
* Submitting the credentials form reboots the ESP32 and lights up the LED widget once connection is established.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `preferences.begin("wifi-creds", true)` | Opens the NVS storage namespace safely in read-only or read-write mode. |
| `preferences.putString("ssid", newSsid)` | Commits the user SSID configuration string to flash memory. |
| `ESP.restart()` | Triggers a hardware watchdog reset to reboot the chip. |
| `WiFi.getMode() == WIFI_AP` | Checks if the device is currently hosting the setup AP. |

## Hardware & Safety Concept: Preferences vs. EEPROM and NVS Flash Wear
* **Preferences vs. EEPROM**: Traditional Arduino uses the `EEPROM` library to write settings. However, the ESP32 does not contain physical EEPROM; it emulates it on flash memory. The **Preferences library** is much better because it structures settings into key-value pairs (using the ESP32 NVS partition) rather than raw addresses, preventing data corruption and configuration misalignment.
* **Flash Wear**: Like SPIFFS, NVS flash sectors have limited write endurance. Since credentials are only saved once during device configuration, this wear is negligible. Do not write dynamic variables (like sensor values or counters) to Preferences in a continuous loop.

## Try This! (Challenges)
1. **SSID scan list**: Add an SSID scanning routine (Project 06) and populate an HTML `<select>` dropdown menu with scanned network names.
2. **Factory Reset Button**: Wire a physical pushbutton (GPIO 14) that formats NVS preferences if held down for more than 5 seconds on boot.
3. **SPIFFS integration**: Log IP connections and reboots to a CSV file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ESP32 loops or gets stuck during reboot | Power supply drop | Reboots cause peak current draws as the WiFi radio boots. Power the ESP32 from a reliable USB port |
| Credentials are not saved on boot | Preferences not closed | Always make sure to call `preferences.end()` to commit settings to flash before restarting the chip |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [08 - ESP32 Access Point mode creation](../beginner/08-esp32-access-point-mode-creation.md)
- [171 - ESP32 Captive Provisioning Portal](171-esp32-captive-provisioning-portal.md)
- [173 - SmartConfig provisioning](173-smartconfig-provisioning.md) (Next project)
