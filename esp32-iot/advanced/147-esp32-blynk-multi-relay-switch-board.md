# 147 - Blynk Multi-relay switch board

Build a cloud-synchronized 4-channel smart relay switch board on the ESP32 that polls Blynk virtual pins V1, V2, V3, and V4 using secure HTTP REST GET requests, and toggles four physical relays on GPIOs 12, 13, 14, and 15.

## Goal
Learn how to poll multiple command variables from cloud databases, parse numerical responses, drive multi-channel relay modules, implement non-blocking schedulers, and build local status web dashboards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Four relays are connected to GPIOs 12, 13, 14, and 15. Every 3 seconds, the ESP32 polls Blynk Virtual Pins V1 to V4. If any virtual pin is set to 1 on the Blynk mobile app dashboard, the ESP32 turns on the corresponding relay. A local webpage displays the current states of the four relays.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4-Channel Relay Module | `relay` | Yes (4 pieces) | Yes (4 pieces) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, monitor the four relay widgets to verify that they open and close in response to cloud command inputs.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay 1 | IN (Signal) | GPIO12 | Yellow | Controls load 1 |
| Relay 2 | IN (Signal) | GPIO13 | Orange | Controls load 2 |
| Relay 3 | IN (Signal) | GPIO14 | Green | Controls load 3 |
| Relay 4 | IN (Signal) | GPIO15 | Blue | Controls load 4 |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the 4-channel relay board from the 5V Vin rail. Many relay boards require 5V to actuate their optocouplers and coils.

## Code
```cpp
// Blynk Multi-relay switch board (Multi-channel Relay Board + Blynk REST client)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Blynk Cloud Auth Token
const String blynkToken = "blynk_token_XYZ123456789";
const String blynkHost = "https://blynk.cloud/external/api/";

// Relay output pins
const int RELAY_PINS[4] = {12, 13, 14, 15};
bool relayStates[4] = {false, false, false, false};

WebServer server(80);

unsigned long lastPollTime = 0;
const unsigned long POLL_INTERVAL_MS = 3000; // Poll Blynk every 3 seconds
String blynkSyncStatus = "Idle";

// Poll a virtual pin from the Blynk cloud server
int pollBlynkPin(String virtualPin) {
  if (WiFi.status() != WL_CONNECTED) return -1;
  
  HTTPClient http;
  String url = blynkHost + "get?token=" + blynkToken + "&" + virtualPin;
  
  http.begin(url);
  int responseCode = http.GET();
  int value = -1;
  
  if (responseCode == 200) {
    String response = http.getString();
    response.trim();
    value = response.toInt(); // Expecting "0" or "1"
  }
  http.end();
  return value;
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"r1\":" + String(relayStates[0] ? 1 : 0) + 
                 ",\"r2\":" + String(relayStates[1] ? 1 : 0) + 
                 ",\"r3\":" + String(relayStates[2] ? 1 : 0) + 
                 ",\"r4\":" + String(relayStates[3] ? 1 : 0) + 
                 ",\"sync\":\"" + blynkSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Relay Hub Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: center; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #ef4444; font-family: monospace; }\n";
  html += "  .metric-val.on { color: #10b981; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Relay Board</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Relay 1</div><div id=\"r1Val\" class=\"metric-val\">OFF</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Relay 2</div><div id=\"r2Val\" class=\"metric-val\">OFF</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Relay 3</div><div id=\"r3Val\" class=\"metric-val\">OFF</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Relay 4</div><div id=\"r4Val\" class=\"metric-val\">OFF</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Blynk Virtual Pins: V1 - V4 | Polling: 3s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateRelays() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  
  for (let i = 1; i <= 4; i++) {
    html += "        const r" + String(i) + "El = document.getElementById('r" + String(i) + "Val');\n";
    html += "        if(data.r" + String(i) + " === 1) {\n";
    html += "          r" + String(i) + "El.innerText = 'ON';\n";
    html += "          r" + String(i) + "El.className = 'metric-val on';\n";
    html += "        } else {\n";
    html += "          r" + String(i) + "El.innerText = 'OFF';\n";
    html += "          r" + String(i) + "El.className = 'metric-val';\n";
    html += "        }\n";
  }
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateRelays();\n";
  html += "    setInterval(updateRelays, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure relay pins as outputs
  for (int i = 0; i < 4; i++) {
    pinMode(RELAY_PINS[i], OUTPUT);
    digitalWrite(RELAY_PINS[i], LOW);
  }
  
  Serial.println("\nESP32 Smart Relay board starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Periodically poll Blynk virtual pins V1 to V4
  unsigned long now = millis();
  if (now - lastPollTime >= POLL_INTERVAL_MS) {
    lastPollTime = now;
    
    Serial.println("[Blynk] Polling relay command pins V1-V4...");
    bool syncSuccess = true;
    
    for (int i = 0; i < 4; i++) {
      String pin = "V" + String(i + 1);
      int state = pollBlynkPin(pin);
      
      if (state != -1) {
        relayStates[i] = (state == 1);
        digitalWrite(RELAY_PINS[i], relayStates[i] ? HIGH : LOW);
      } else {
        syncSuccess = false;
        Serial.printf("[Blynk Error] Failed to read pin: %s\n", pin.c_str());
      }
    }
    
    if (syncSuccess) {
      blynkSyncStatus = "Success";
      Serial.printf("[Relays] States synced -> R1: %s | R2: %s | R3: %s | R4: %s\n",
                    relayStates[0] ? "ON" : "OFF", relayStates[1] ? "ON" : "OFF",
                    relayStates[2] ? "ON" : "OFF", relayStates[3] ? "ON" : "OFF");
    } else {
      blynkSyncStatus = "Sync Error";
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **4 Relays** onto the canvas.
2. Wire Relay inputs to **GPIO12, GPIO13, GPIO14, and GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, verify that the four relay widgets are open (OFF).
6. Verify that the Serial Monitor outputs `Polling relay command pins V1-V4...` every 3 seconds.
7. Mock the network response by inspecting how the code processes incoming virtual pin states.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Blynk] Polling relay command pins V1-V4...
[Relays] States synced -> R1: ON | R2: OFF | R3: ON | R4: OFF
```

## Expected Canvas Behavior
* The simulated relay widgets open or close automatically based on parsed virtual pin values.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `RELAY_PINS[i]` | Maps indices to output GPIOs driving the relays. |
| `pollBlynkPin(pin)` | Requests the state of a specific virtual pin from the Blynk server. |
| `digitalWrite(RELAY_PINS[i], ...)` | Updates the output state of the physical relay pins. |

## Hardware & Safety Concept: Command Batching and High Voltage Isolation
* **Command Batching**: Polling four separate cloud endpoints (`V1` to `V4`) sequentially creates significant network overhead (taking up to 2 seconds). In hardware systems, it is better to query a single JSON endpoint containing all pin values (e.g. `https://blynk.cloud/external/api/get?token=[token]&V1&V2&V3&V4`) to fetch all states in a single GET request, reducing traffic and network latency.
* **High Voltage Safety**: Relays are used to switch AC mains loads (110V/220V). Ensure that the PCB traces for the high-voltage relay contacts are physically isolated from the low-voltage DC logic circuits using isolation slots (air gaps) to prevent electric arcs from jumping and destroying the ESP32.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a 2x2 grid representing the states of the relays.
2. **Physical Overrides**: Add four pushbuttons (GPIOs 32, 33, 25, 26) to allow manual override toggles of the relays locally.
3. **SPIFFS State Saver**: Save the current relay states to SPIFFS so the board recovers its previous state after a power outage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relays click rapidly during boot | Floating pins | Ensure output pins are explicitly declared as `OUTPUT` and set to `LOW` in the `setup()` function |
| Polling returns -1 | Invalid token | Double check the Blynk Auth Token in the code |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [146 - ESP32 Blynk IoT Smart Home Hub](146-esp32-blynk-iot-smart-home-hub.md)
- [148 - IFTTT Alarm Notification node](148-esp32-ifttt-alarm-notification-node-gas-leak-trigger-email-alert.md) (Next project)
