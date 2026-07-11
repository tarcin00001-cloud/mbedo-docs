# 145 - Adafruit IO Remote Actuator controller

Build a cloud-controlled actuator node on the ESP32 that polls command feeds from an Adafruit IO dashboard using secure HTTP REST GET requests, and toggles a physical LED on GPIO 12 and a relay on GPIO 13.

## Goal
Learn how to fetch actuator states from Adafruit IO, parse JSON key-value response fields, control digital output pins, implement periodic polling timers, and host local status overrides.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LED is on GPIO 12, and a relay on GPIO 13. Every 5 seconds, the ESP32 requests the latest data points from two Adafruit IO feeds (`led-switch` and `relay-switch`). If the dashboard buttons are toggled ON, the ESP32 turns on the matching LED or relay. It also hosts a local status page to inspect connection diagnostics.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Status indicator output |
| Relay Module | IN (Signal) | GPIO13 | Blue | Output relay switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED anode. Power the relay from the 5V Vin rail.

## Code
```cpp
// Adafruit IO Remote Actuator controller (Adafruit IO REST Feed Poller + Actuators)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Adafruit IO credentials
const String aioUsername = "your_username";
const String aioKey = "aio_XYZ123456789"; // Place your Adafruit IO key here

const int LED_PIN = 12;
const int RELAY_PIN = 13;

WebServer server(80);

// Global actuator states
bool ledState = false;
bool relayState = false;

unsigned long lastPollTime = 0;
const unsigned long POLL_INTERVAL_MS = 5000; // Poll Adafruit IO every 5 seconds
String adafruitSyncStatus = "Idle";

// Poll the latest data point from a specific Adafruit IO feed
String pollFeedLastValue(String feedKey) {
  if (WiFi.status() != WL_CONNECTED) return "ERROR";
  
  HTTPClient http;
  
  // Endpoint format: https://io.adafruit.com/api/v2/[username]/feeds/[feed_key]/data/last
  String url = "https://io.adafruit.com/api/v2/" + aioUsername + "/feeds/" + feedKey + "/data/last";
  
  http.begin(url);
  http.addHeader("X-AIO-Key", aioKey); // Authenticate using X-AIO-Key header
  
  int responseCode = http.GET();
  String value = "OFF";
  
  if (responseCode == 200) {
    String response = http.getString();
    
    // Very simple JSON parsing for value field
    // response format contains: "value":"ON" or "value":"1"
    int valueIdx = response.indexOf("\"value\":\"");
    if (valueIdx != -1) {
      int endIdx = response.indexOf("\"", valueIdx + 9);
      if (endIdx != -1) {
        value = response.substring(valueIdx + 9, endIdx);
      }
    }
  } else {
    value = "ERROR";
    Serial.printf("[Adafruit IO] Error polling feed %s: %d\n", feedKey.c_str(), responseCode);
  }
  
  http.end();
  return value;
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"led\":" + String(ledState ? 1 : 0) + 
                 ",\"relay\":" + String(relayState ? 1 : 0) + 
                 ",\"sync\":\"" + adafruitSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Adafruit Actuator HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: center; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .metric-val.off { color: #ef4444; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Adafruit Actuator HUD</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">LED state</div><div id=\"ledVal\" class=\"metric-val off\">OFF</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Relay state</div><div id=\"relayVal\" class=\"metric-val off\">OFF</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Feeds polled: led-switch, relay-switch | Interval: 5s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateActuators() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const ledEl = document.getElementById('ledVal');\n";
  html += "        if(data.led === 1) {\n";
  html += "          ledEl.innerText = 'ON';\n";
  html += "          ledEl.className = 'metric-val';\n";
  html += "        } else {\n";
  html += "          ledEl.innerText = 'OFF';\n";
  html += "          ledEl.className = 'metric-val off';\n";
  html += "        }\n";
  
  html += "        const relayEl = document.getElementById('relayVal');\n";
  html += "        if(data.relay === 1) {\n";
  html += "          relayEl.innerText = 'ON';\n";
  html += "          relayEl.className = 'metric-val';\n";
  html += "        } else {\n";
  html += "          relayEl.innerText = 'OFF';\n";
  html += "          relayEl.className = 'metric-val off';\n";
  html += "        }\n";
  
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
  html += "    updateActuators();\n";
  html += "    setInterval(updateActuators, 1000); // Local poll index\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
  
  Serial.println("\nESP32 Actuator Station Starting...");
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
  
  // Periodically query Adafruit IO for changes
  unsigned long now = millis();
  if (now - lastPollTime >= POLL_INTERVAL_MS) {
    lastPollTime = now;
    
    Serial.println("[Adafruit IO] Polling command feeds...");
    String ledCmd = pollFeedLastValue("led-switch");
    String relayCmd = pollFeedLastValue("relay-switch");
    
    if (ledCmd != "ERROR" && relayCmd != "ERROR") {
      adafruitSyncStatus = "Success";
      
      // Update LED pin
      if (ledCmd == "ON" || ledCmd == "1") {
        ledState = true;
        digitalWrite(LED_PIN, HIGH);
      } else {
        ledState = false;
        digitalWrite(LED_PIN, LOW);
      }
      
      // Update Relay pin
      if (relayCmd == "ON" || relayCmd == "1") {
        relayState = true;
        digitalWrite(RELAY_PIN, HIGH);
      } else {
        relayState = false;
        digitalWrite(RELAY_PIN, LOW);
      }
      
      Serial.printf("[Actuators] Synced -> LED: %s | Relay: %s\n", 
                    ledState ? "ON" : "OFF", relayState ? "ON" : "OFF");
    } else {
      adafruitSyncStatus = "Polling Error";
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **LED**, and **Relay** onto the canvas.
2. Wire LED to **GPIO12** and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, verify that both the LED and relay are OFF.
6. Verify that the Serial Monitor outputs `Polling command feeds...` every 5 seconds.
7. Mock the network response by inspecting how the code handles toggled strings to drive output widgets.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Adafruit IO] Polling command feeds...
[Actuators] Synced -> LED: ON | Relay: OFF
```

## Expected Canvas Behavior
* The simulated LED and relay widgets toggle state based on parsed command inputs from the web client.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pollFeedLastValue(...)` | Sends the HTTP GET request to check the latest value of a feed. |
| `response.indexOf("\"value\":\"")` | Searches the response text for the data value tag. |
| `digitalWrite(LED_PIN, ...)` | Sets the GPIO output pin to drive the actuator. |

## Hardware & Safety Concept: Actuator Safe-States on Network Failures
* **Actuator Safe-States**: When building remote-controlled hardware (like heaters, valves, or motor drives), network interruptions are common. If the ESP32 loses connection, it will stop polling the cloud dashboard. If a heater was left ON, it could cause overheating. Always implement a safety timeout: if the device fails to poll the server for more than 30 seconds, it should automatically turn off all high-power actuators (`digitalWrite(RELAY_PIN, LOW)`) to ensure a safe shutdown state.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and show status icons for the LED and relay.
2. **Alert sound**: Add a buzzer (GPIO 15) to chirp when an actuator state changes.
3. **Adafruit IO MQTT subscribe**: Implement the Adafruit IO MQTT subscription interface to receive actuator commands instantly rather than polling.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Actuators do not toggle | Invalid AIO key or username | Verify that the credentials in the code match your Adafruit IO account |
| Actuator response has long delay | Polling interval too slow | The polling interval is set to 5 seconds. You can reduce it to 3 seconds, but be careful of rate limits |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [144 - Adafruit IO Dashboard logger](144-esp32-adafruit-io-dashboard-logger.md)
- [146 - Blynk IoT Smart Home Hub](146-esp32-blynk-iot-smart-home-hub.md) (Next project)
