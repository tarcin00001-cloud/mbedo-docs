# 175 - SSDP Device Discovery (Advertises device on Windows/UPnP network)

Build a local network discovery node on the ESP32 that runs the Simple Service Discovery Protocol (SSDP) over UPnP, allowing Windows computers to automatically discover and list the ESP32 under the file explorer's "Network" folder for easy navigation.

## Goal
Learn how to implement Simple Service Discovery Protocol (SSDP) logic, construct UPnP XML configuration files, register network device schemas, and handle presentation redirection requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An HTTP server is started on port 80. The ESP32 runs the SSDP responder service, advertising its friendly name `ESP32 Smart Device` and serving an XML device schema at `/ssdp/schema.xml`. When a Windows computer searches the network, it discovers the ESP32 and displays a custom icon. Double-clicking this icon opens the ESP32's home page in the web browser.

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

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to prevent overloading the pin.

## Code
```cpp
// SSDP Device Discovery (Simple Service Discovery Protocol + UPnP XML + Web status)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32SSDP.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 12;

WebServer server(80);

// Serve main page
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>SSDP Smart Node</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 420px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #065f46; color: #a7f3d0; }\n";
  html += "  p { color: #94a3b8; font-size: 15px; line-height: 1.6; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>SSDP Device Discovery</h1>\n";
  html += "  <div class=\"badge\">SSDP ADVERTISING</div>\n";
  html += "  <p>This ESP32 is broadcasting UPnP SSDP configuration signals. Open Windows File Explorer and check the 'Network' folder to discover it automatically.</p>\n";
  html += "  <p class=\"footer\">Device Serial: ESP32-123456 | Schema: /ssdp/schema.xml</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, HIGH); // LED indicates power
  
  Serial.println("\nESP32 SSDP Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // 1. Configure SSDP parameters
  SSDP.setSchemaURL("ssdp/schema.xml");
  SSDP.setHTTPPort(80);
  SSDP.setName("ESP32 Smart Device");
  SSDP.setSerialNumber("ESP32-123456");
  SSDP.setURL("/");
  SSDP.setModelName("MbedO ESP32 IoT Node");
  SSDP.setModelNumber("V1");
  SSDP.setManufacturer("Espressif");
  SSDP.setDeviceType("urn:schemas-upnp-org:device:Basic:1");
  
  // 2. Start SSDP responder
  if (!SSDP.begin()) {
    Serial.println("[SSDP Error] Failed to start SSDP responder!");
  } else {
    Serial.println("[SSDP] Responder active. Schema: http://<IP>/ssdp/schema.xml");
  }
  
  // 3. Set up Web Server routes
  server.on("/", handleRoot);
  
  // SSDP library writes device schema directly to client context
  server.on("/ssdp/schema.xml", []() {
    SSDP.schema(server.client());
  });
  
  server.begin();
  Serial.println("HTTP Server active.");
}

void loop() {
  server.handleClient();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open a new browser tab and navigate to `http://10.10.0.3/ssdp/schema.xml`.
5. Verify that the browser displays the valid UPnP XML schema layout containing manufacturer and model properties.
6. Open Windows Explorer on your host machine, select **Network**, and verify that `ESP32 Smart Device` appears in the list of local network devices.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[SSDP] Responder active. Schema: http://<IP>/ssdp/schema.xml
HTTP Server active.
```

Browser XML Output (`/ssdp/schema.xml`):
```xml
<root xmlns="urn:schemas-upnp-org:device-1-0">
  <specVersion><major>1</major><minor>0</minor></specVersion>
  <device>
    <deviceType>urn:schemas-upnp-org:device:Basic:1</deviceType>
    <friendlyName>ESP32 Smart Device</friendlyName>
    <presentationURL>/</presentationURL>
    <serialNumber>ESP32-123456</serialNumber>
    <modelName>MbedO ESP32 IoT Node</modelName>
    <manufacturer>Espressif</manufacturer>
  </device>
</root>
```

## Expected Canvas Behavior
* Running the code triggers the SSDP broadcast loop. Navigating to the schema URL returns the XML metadata structure.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SSDP.setSchemaURL(...)` | Sets the web location where the UPnP XML schema is served. |
| `SSDP.setDeviceType(...)` | Sets the UPnP device category type. |
| `SSDP.begin()` | Starts listening for M-SEARCH UDP multicast discovery broadcasts. |
| `SSDP.schema(server.client())` | Serializes and writes the XML schema configuration to the client browser. |

## Hardware & Safety Concept: UPnP UDP Multicast Ports and Network Security
* **SSDP UDP Multicast**: SSDP communicates using UDP multicast packets on port 1900 to the multicast IP `239.255.255.250`. When a host computer broadcasts a discovery packet (`M-SEARCH`), the ESP32 intercepts the packet and responds with its IP address and schema URL.
* **Network Security**: SSDP has no encryption or access control features. If the network is compromised, attackers can use SSDP reflection attacks to flood devices with garbage data. For high-security installations, disable SSDP and mDNS broadcasts, relying instead on static IP configurations and secure HTTPS connections.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "SSDP Broadcast Active" on boot.
2. **Dynamic Name Update**: Modify the code to change the SSDP name dynamically based on user settings saved in NVS (Project 172).
3. **Control led brightness**: Add a slider to control the brightness of an LED (GPIO 12) via PWM from the main dashboard.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Device not showing in Windows Network | Network discovery disabled | Open Windows settings, navigate to "Network and Sharing Center", click "Change advanced sharing settings", and enable Network Discovery |
| Schema fails to load | Mismatched XML headers | Ensure the response content-type is set correctly and the XML tags match exactly |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [12 - ESP32 DNS hostname configuration](../beginner/12-esp32-dns-hostname-configuration.md)
- [174 - ESP32 mDNS local domain hostname setup](174-mdns-local-domain-hostname-setup.md)
- [176 - LLMNR Hostname configuration](176-llmnr-hostname-configuration.md) (Next project)
