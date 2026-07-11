# 179 - ESP-NOW to WiFi Gateway Hub (receives ESP-NOW packets and uploads to web)

Build a wireless gateway router node on the ESP32 that runs ESP-NOW and WiFi STA modes concurrently on the same channel, intercepts data packets sent by remote low-power sensor nodes, flashes a status sync LED on GPIO 12, and bridges the parameters to a cloud database using HTTP POST.

## Goal
Learn how to configure ESP-NOW and WiFi STA coexistence architectures, manage incoming ESP-NOW packet structures, execute asynchronous HTTP requests, and host diagnostic local web interfaces.

## What You Will Build
An ESP32 DevKitC acts as a gateway hub. It connects to a local WiFi network (which sets the radio channel). It then initializes the ESP-NOW protocol on that same channel. When a remote sensor node sends an ESP-NOW packet containing its name and reading, the gateway intercepts the packet, blinks the LED on GPIO 12, uploads a JSON packet to a cloud service (`http://httpbin.org/post`), and updates its local webpage database.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes (2 pieces to demonstrate bridge) |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, remote ESP-NOW nodes can be simulated using virtual network packets or simulated serial injections.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Data sync indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// ESP-NOW to WiFi Gateway Hub (Dual Coexistence Mode + HTTP Client + Web Log Dashboard)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <esp_now.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int SYNC_LED = 12;

WebServer server(80);

// ESP-NOW Data Structure (Must match sender node struct)
typedef struct struct_message {
  char sensorName[16];
  float reading;
  uint32_t counter;
} struct_message;

struct_message incomingData;

// Telemetry cache
String nodeName = "No Node";
float nodeReading = 0.0;
uint32_t nodeCounter = 0;
String cloudBridgeStatus = "Idle";
unsigned long lastUploadTime = 0;

// Flag to trigger background HTTP upload
volatile bool dataReceivedFlag = false;

// Callback function executed when ESP-NOW data is received
void onDataRecv(const uint8_t * mac, const uint8_t *incoming, int len) {
  memcpy(&incomingData, incoming, sizeof(incomingData));
  
  nodeName = String(incomingData.sensorName);
  nodeReading = incomingData.reading;
  nodeCounter = incomingData.counter;
  
  // Set flag to process HTTP POST in main loop (do not perform HTTP in ISR context)
  dataReceivedFlag = true;
}

// Upload bridged data to cloud database
void uploadDataToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    // Blink LED to indicate active bridge processing
    digitalWrite(SYNC_LED, HIGH);
    
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"sensor\":\"" + nodeName + "\"" +
                         ",\"value\":" + String(nodeReading, 2) + 
                         ",\"packetId\":" + String(nodeCounter) + 
                         ",\"gatewayUptime\":" + String(millis() / 1000) + "}";
                         
    Serial.printf("[Gateway Bridge] Forwarding payload: %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      cloudBridgeStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.println("[Gateway Bridge] Cloud upload successful.");
    } else {
      cloudBridgeStatus = "Failed (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Gateway Error] POST failed: %d\n", httpResponseCode);
    }
    http.end();
    
    delay(100); // Hold LED on briefly
    digitalWrite(SYNC_LED, LOW);
  } else {
    cloudBridgeStatus = "WiFi Offline";
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"node\":\"" + nodeName + "\"" +
                 ",\"reading\":" + String(nodeReading, 2) + 
                 ",\"counter\":" + String(nodeCounter) + 
                 ",\"bridge\":\"" + cloudBridgeStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP-NOW Gateway HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .reading-display { font-size: 48px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 20px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.active { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>ESP-NOW Gateway</h1>\n";
  html += "  <div id=\"bridgeBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div id=\"readDisplay\" class=\"reading-display\">0.00</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Active Node</div><div class=\"metric-val\" id=\"nodeDisplay\">None</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Packet Count</div><div class=\"metric-val\" id=\"countDisplay\">0</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">WiFi Channel: Locked by Router | Target: httpbin.org</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateGatewayHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('readDisplay').innerText = data.reading.toFixed(2);\n";
  html += "        document.getElementById('nodeDisplay').innerText = data.node;\n";
  html += "        document.getElementById('countDisplay').innerText = data.counter;\n";
  
  html += "        const bdgEl = document.getElementById('bridgeBadge');\n";
  html += "        bdgEl.innerText = data.bridge;\n";
  html += "        if (data.bridge.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge active';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateGatewayHUD();\n";
  html += "    setInterval(updateGatewayHUD, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(SYNC_LED, OUTPUT);
  digitalWrite(SYNC_LED, LOW);
  
  // 1. Initialize WiFi in Station mode
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  Serial.print("Connecting to local WiFi network...");
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  Serial.print("Gateway IP: ");
  Serial.println(WiFi.localIP());
  
  // 2. Initialize ESP-NOW
  // Coexistence mode: ESP-NOW automatically inherits the WiFi channel used by the router
  if (esp_now_init() != ESP_OK) {
    Serial.println("[ESP-NOW Error] Failed to initialize protocol!");
    return;
  }
  
  // Register receive callback
  esp_now_register_recv_cb(onDataRecv);
  Serial.println("[ESP-NOW] Protocol active and listening on WiFi channel.");
  
  // 3. Set up Web Server routes
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.onNotFound([]() { server.send(404, "text/plain", "Not Found"); });
  server.begin();
  
  Serial.println("HTTP Diagnostic Server active.");
}

void loop() {
  server.handleClient();
  
  // 4. Process ESP-NOW uploads asynchronously in loop context
  if (dataReceivedFlag) {
    dataReceivedFlag = false; // Reset flag
    uploadDataToCloud();
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** (acting as Gateway Hub) and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, since a second ESP-NOW physical device is not present, we simulate packet reception via serial commands. Enter the text string `SYS:NODE1:24.5` in the serial terminal.
5. Verify that the LED on the canvas flashes once, and the console shows the payload forwarding message.
6. Open your web browser and navigate to the printed IP address.
7. Verify that the webpage active node displays `NODE1`, the value updates to `24.50`, and the sync status displays `Success`.

## Expected Output
Serial Monitor:
```
WiFi Connected successfully.
Gateway IP: 10.10.0.3
[ESP-NOW] Protocol active and listening on WiFi channel.
HTTP Diagnostic Server active.
[Gateway Bridge] Forwarding payload: {"sensor":"NODE1","value":24.50,"packetId":1,"gatewayUptime":15}
[Gateway Bridge] Cloud upload successful.
```

## Expected Canvas Behavior
* Sending simulated serial packets triggers the sync LED widget and updates the active web indicators.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `esp_now_init()` | Starts the ESP-NOW communication interface in coexistence mode. |
| `esp_now_register_recv_cb(...)` | Registers the interrupt-level callback executed when a packet is received. |
| `dataReceivedFlag = true` | Safety flag setting execution of HTTP POST outside of ISR contexts. |
| `http.POST(jsonPayload)` | Forwards the decoded sensor data payload to the cloud API. |

## Hardware & Safety Concept: Coexistence Channel Locking and ISR Contexts
* **Coexistence Channel Locking**: The ESP32 contains a single physical 2.4 GHz radio receiver. In STA mode, the ESP32 locks onto the channel used by your home router (e.g. Channel 6). If the remote ESP-NOW nodes transmit on a different channel (e.g. Channel 1), the gateway will not hear them. Ensure all remote transmitter nodes query the current router channel before initializing their ESP-NOW broadcasts.
* **ISR Contexts**: The ESP-NOW receive callback is executed at the interrupt level (ISR). Web client operations (like DNS lookups and HTTP POST requests) take hundreds of milliseconds and can trigger watchdog resets if run inside an ISR. Always set a volatile boolean flag in the callback and process the web request in the main `loop()` context.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist of connected nodes.
2. **Double Channel Fail-Safe**: Modify the code to trigger an alert sound on a buzzer (GPIO 15) if no ESP-NOW packet is received from a critical node for more than 5 minutes (node offline warning).
3. **SPIFFS integration**: Log all bridged packets to a CSV file in SPIFFS (Project 165).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No packets are received from transmitter | Channel mismatch | Ensure the transmitter node is configured to use the same WiFi channel as the gateway hub. Read the channel from the router connection on the gateway and set it on the transmitter |
| ESP32 crashes on packet arrival | HTTP inside callback | Ensure that you are not executing any HTTP or delay operations inside the `onDataRecv` callback function |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [90 - ESP-NOW Peer-to-Peer communications](../intermediate/90-esp-now-peer-to-peer-communications.md)
- [120 - Bluetooth to WiFi Telemetry Gateway](../intermediate/120-bluetooth-to-wifi-telemetry-gateway.md)
- [180 - SPIFFS Web File Manager](180-spiffs-web-file-manager.md) (Next project)
