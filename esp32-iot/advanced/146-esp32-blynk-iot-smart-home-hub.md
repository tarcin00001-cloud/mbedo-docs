# 146 - Blynk IoT Smart Home Hub (Blynk mobile app connection)

Build a smartphone-integrated smart home hub on the ESP32 that connects to the Blynk IoT Cloud platform using secure HTTP REST API endpoints to upload DHT22 temperature readings to Virtual Pin V1 and poll Virtual Pin V2 to toggle a physical LED on GPIO 12.

## Goal
Learn how to use the Blynk Cloud REST API, construct URL query strings, handle status updates, process incoming commands via polling loops, and manage output actuators.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and an LED on GPIO 12. Every 10 seconds, the ESP32 uploads the current temperature to Blynk Virtual Pin V1 (`https://blynk.cloud/external/api/update?token=[token]&V1=[temp]`). Simultaneously, it polls Virtual Pin V2 to check for commands from the Blynk mobile app, turning the LED on or off accordingly. A local webpage displays live statuses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| LED | Anode (+) | GPIO12 | Red | Status indicator output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED anode. Power the DHT22 from the 3.3V rail.

## Code
```cpp
// Blynk IoT Smart Home Hub (DHT22 sensor + Blynk REST client + LED control)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Blynk Cloud Auth Token
// Get this token from your Blynk Developer Console dashboard device settings
const String blynkToken = "blynk_token_XYZ123456789";
const String blynkHost = "https://blynk.cloud/external/api/";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int LED_PIN = 12;

WebServer server(80);

// Global values
float currentTemp = 0.0;
bool ledState = false;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 10000; // Log data every 10 seconds
String blynkSyncStatus = "Idle";

// Update Blynk Virtual Pin V1 with Temperature
bool uploadToBlynk(String virtualPin, float value) {
  if (WiFi.status() != WL_CONNECTED) return false;
  
  HTTPClient http;
  
  // Format: https://blynk.cloud/external/api/update?token=[token]&[pin]=[val]
  String url = blynkHost + "update?token=" + blynkToken + "&" + virtualPin + "=" + String(value, 1);
  
  http.begin(url);
  int responseCode = http.GET();
  http.end();
  
  return (responseCode == 200);
}

// Poll Blynk Virtual Pin V2 for LED State Command
int pollBlynkPin(String virtualPin) {
  if (WiFi.status() != WL_CONNECTED) return -1;
  
  HTTPClient http;
  
  // Format: https://blynk.cloud/external/api/get?token=[token]&[pin]
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
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"led\":" + String(ledState ? 1 : 0) + 
                 ",\"sync\":\"" + blynkSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Blynk Hub Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Blynk Smart Home Hub</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">LED State</div><div class=\"metric-val\" id=\"ledDisplay\">OFF</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Blynk Pins: V1 (Temp Out), V2 (LED In) | Interval: 10s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateBlynkHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('ledDisplay').innerText = data.led === 1 ? 'ON' : 'OFF';\n";
  
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
  html += "    updateBlynkHUD();\n";
  html += "    setInterval(updateBlynkHUD, 2000);\n";
  // Handled network updates safely
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 Blynk Hub starting...");
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
  
  // Refresh temperature value
  currentTemp = dht.readTemperature();
  
  // Periodically synchronize with Blynk Cloud (every 10 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    
    if (!isnan(currentTemp)) {
      Serial.println("[Blynk] Synchronizing cloud database...");
      bool tOk = uploadToBlynk("V1", currentTemp);
      
      // Poll command pin
      int ledCmd = pollBlynkPin("V2");
      
      if (tOk && ledCmd != -1) {
        blynkSyncStatus = "Success";
        
        // Update physical LED
        if (ledCmd == 1) {
          ledState = true;
          digitalWrite(LED_PIN, HIGH);
        } else {
          ledState = false;
          digitalWrite(LED_PIN, LOW);
        }
        
        Serial.printf("[Blynk] Sync completed -> Temperature: %.1f C | LED: %s\n", 
                      currentTemp, ledState ? "ON" : "OFF");
      } else {
        blynkSyncStatus = "Sync Error";
        Serial.println("[Blynk] Failed to complete sync cycle.");
      }
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **LED** onto the canvas.
2. Wire DHT22 to **GPIO14** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, verify that the webpage displays temperature and LED state.
6. Verify that the Serial Monitor outputs `Synchronizing cloud database...` every 10 seconds.
7. Mock the network response by inspecting how the code processes incoming virtual pin data.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Blynk] Synchronizing cloud database...
[Blynk] Sync completed -> Temperature: 24.0 C | LED: ON
```

## Expected Canvas Behavior
* The simulated LED widget toggles state based on virtual pin commands retrieved from the cloud database connection.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `uploadToBlynk("V1", currentTemp)` | Pushes the temperature reading to Blynk Virtual Pin V1. |
| `pollBlynkPin("V2")` | Requests the latest value of Virtual Pin V2 from the Blynk server. |
| `digitalWrite(LED_PIN, ...)` | Sets the GPIO output pin to drive the physical LED. |

## Hardware & Safety Concept: Cloud Connection Timeouts and Fail-Safe Outputs
* **Connection Timeouts**: Connecting to cloud APIs requires DNS lookups and secure handshakes. If the cloud servers are slow or offline, HTTP requests can freeze execution for up to 15 seconds. Ensure that you configure reasonable HTTP connection timeouts (`http.setTimeout(3000)`) in your production code to prevent freezing local control loops.
* **Fail-Safe Outputs**: Actuators controlled by mobile apps must have local safety check limits. For example, if the app sends a command to turn on a heater relay, but the local temperature sensor reads a dangerous value, the local code should override the cloud command and shut down the relay.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a phone sync icon when Blynk updates.
2. **Alert sound**: Add a buzzer (GPIO 15) to play a chime when an app command is received.
3. **Blynk Multi-Pin Actuators**: Add an additional virtual pin polling check for a relay (GPIO 13).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Cloud upload fails with code -1 | Invalid Blynk token | Verify that you copied the correct authentication token from the Blynk console |
| LED command is ignored | Virtual pin wrong | Confirm that your Blynk mobile app widget is bound to Virtual Pin V2 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [142 - Firebase Realtime Database Logger](142-firebase-realtime-database-logger-upload-current-state-to-firebase.md)
- [145 - Adafruit IO Remote Actuator controller](145-esp32-adafruit-io-remote-actuator-controller.md)
- [147 - Blynk Multi-relay switch board](147-esp32-blynk-multi-relay-switch-board.md) (Next project)
