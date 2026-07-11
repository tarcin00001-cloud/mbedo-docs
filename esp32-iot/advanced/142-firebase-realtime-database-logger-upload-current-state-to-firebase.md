# 142 - Firebase Realtime Database Logger (Upload current state to Firebase)

Build a real-time IoT database logger on the ESP32 that samples a DHT22 climate sensor on GPIO 14, monitors LED and relay status states (on GPIOs 12 and 13), pushes these states to Google's Firebase Realtime Database via HTTP REST requests, and hosts a local dashboard.

## Goal
Learn how to use the Firebase Database REST API, compile HTTP PATCH requests, serialize status states into nested JSON payloads, and manage secure cloud uploads without SDK dependencies.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It reads temperature and humidity from a DHT22 sensor, and tracks the states of an LED and a relay. Every 10 seconds, the ESP32 sends an HTTP PATCH request containing a JSON payload of these values to a Firebase database simulator (`https://esp32-default-rtdb.firebaseio.com/state.json`). A local webpage displays the latest values and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Relay Module | IN (Signal) | GPIO13 | Blue | Output relay switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED anode to limit current. Power the relay from the 5V Vin rail.

## Code
```cpp
// Firebase Realtime Database Logger (DHT22 sensor + Firebase REST client)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Firebase Realtime Database REST URL
// Format: https://[your-project-id]-default-rtdb.firebaseio.com/[path].json
const String firebaseHost = "https://esp32-default-rtdb.firebaseio.com/state.json";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int LED_PIN = 12;
const int RELAY_PIN = 13;

WebServer server(80);

// Global values
float currentTemp = 0.0;
float currentHumid = 0.0;
bool ledState = false;
bool relayState = false;

unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 10000; // Upload state every 10 seconds
String firebaseSyncStatus = "Idle";

// Send HTTP PATCH request to update Firebase RTDB
void uploadToFirebase() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // We send to firebaseHost URL
    // Note: Firebase REST requires HTTPS. For simplicity, we bypass SSL verification in HTTPClient.
    http.begin(firebaseHost);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload containing nested variables
    String jsonPayload = "{\"temp\":" + String(currentTemp, 1) + 
                         ",\"humidity\":" + String(currentHumid, 1) + 
                         ",\"led\":" + String(ledState ? "true" : "false") + 
                         ",\"relay\":" + String(relayState ? "true" : "false") + 
                         ",\"uptime\":" + String(millis() / 1000) + "}";
                         
    Serial.printf("[Firebase] Pushing PATCH payload -> %s\n", jsonPayload.c_str());
    
    // Send PATCH request to merge data with existing database node
    int httpResponseCode = http.PATCH(jsonPayload);
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      firebaseSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Firebase] Sync successful. Code: %d\n", httpResponseCode);
    } else {
      firebaseSyncStatus = "Failed (Error: " + String(httpResponseCode) + ")";
      Serial.printf("[Firebase] Error sending PATCH request: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    firebaseSyncStatus = "WiFi Offline";
    Serial.println("[Firebase] Sync blocked. WiFi offline.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"humidity\":" + String(currentHumid, 1) + 
                 ",\"led\":" + String(ledState ? 1 : 0) + 
                 ",\"relay\":" + String(relayState ? 1 : 0) + 
                 ",\"sync\":\"" + firebaseSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to toggle local devices
void handlePostToggle() {
  if (server.hasArg("device")) {
    String device = server.arg("device");
    if (device == "led") {
      ledState = !ledState;
      digitalWrite(LED_PIN, ledState ? HIGH : LOW);
    } else if (device == "relay") {
      relayState = !relayState;
      digitalWrite(RELAY_PIN, relayState ? HIGH : LOW);
    }
    // Upload state change immediately
    uploadToFirebase();
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Firebase Logger HUD</title>\n";
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
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 15px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.active { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Firebase Logger HUD</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humidDisplay\">--.- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <h3>Device Controls</h3>\n";
  html += "  <div class=\"btn-group\">\n";
  html += "    <button id=\"btnLed\" class=\"btn\" onclick=\"toggleDevice('led')\">Toggle LED</button>\n";
  html += "    <button id=\"btnRelay\" class=\"btn\" onclick=\"toggleDevice('relay')\">Toggle Relay</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Firebase Endpoint: state.json | Upload: 10s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateWeather() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humidDisplay').innerText = data.humidity.toFixed(1) + ' %';\n";
  
  html += "        document.getElementById('btnLed').className = 'btn' + (data.led === 1 ? ' active' : '');\n";
  html += "        document.getElementById('btnRelay').className = 'btn' + (data.relay === 1 ? ' active' : '');\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  // If it's a redirection or local error, show as normal warning
  html += "      });\n";
  html += "  }\n";
  
  html += "  function toggleDevice(devName) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('device', devName);\n";
  html += "    fetch('/api/toggle', { method: 'POST', body: body })\n";
  html += "      .then(() => updateWeather());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateWeather();\n";
  html += "    setInterval(updateWeather, 2000);\n";
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
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
  
  Serial.println("\nESP32 Firebase REST node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/toggle", HTTP_POST, handlePostToggle);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Refresh environmental variables
  currentTemp = dht.readTemperature();
  currentHumid = dht.readHumidity();
  
  // Periodically upload data to Firebase (every 10 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    if (!isnan(currentTemp) && !isnan(currentHumid)) {
      uploadToFirebase();
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, **LED**, and **Relay** onto the canvas.
2. Wire DHT22 to **GPIO14**, LED to **GPIO12**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the "Toggle LED" button on the webpage. Verify that the simulated LED widget turns ON/OFF, and the console logs a payload upload to Firebase.
6. Verify that the sync status badge updates to success.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Firebase] Pushing PATCH payload -> {"temp":24.0,"humidity":40.0,"led":true,"relay":false,"uptime":12}
[Firebase] Sync successful. Code: 200
```

## Expected Canvas Behavior
* Clicking the toggle buttons on the webpage turns the simulated LED and relay widgets ON/OFF immediately.
* The web console displays the synchronization status.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples the temperature in Celsius from the DHT22. |
| `http.PATCH(jsonPayload)` | Sends an HTTP PATCH request to update the database values without overwriting other nodes. |
| `firebaseHost` | Path to the target JSON node in the Firebase Realtime Database. |

## Hardware & Safety Concept: REST SSL Verification and Firebase Security Rules
* **SSL Verification**: Secure endpoints (HTTPS) require verify checking of SSL certificates to prevent man-in-the-middle attacks. For production devices, implement cert pinning or use root certificates in `WiFiClientSecure` to protect database secrets.
* **Database Security Rules**: By default, Firebase databases allow public read/write access for testing, which is insecure. Secure production databases by setting database rules and adding authentication tokens (`?auth=SECRET_KEY`) to REST URLs.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a Firebase logo icon.
2. **Alert sound**: Add a buzzer (GPIO 15) to sound a chime when a sync succeeds.
3. **Database Listener**: Use WebSockets or Server-Sent Events (SSE) to listen for database changes in real time.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sync fails with status code -1 | DNS or SSL issue | Ensure that the ESP32 can connect to Google hosts |
| Firebase returns 401 Unauthorized | Database rules blocking | Check that your Firebase database rules allow read/write access |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [140 - ESP32 ThingSpeak Environment Monitor](140-esp32-thingspeak-environment-monitor-upload-temp-humidity-to-thingspeak-channel.md)
- [141 - ESP32 ThingSpeak Multi-Field Sensor database](141-esp32-thingspeak-multi-field-sensor-database-pressure-temp-gas.md)
- [143 - Firebase Authentication Access Controller](143-firebase-authentication-access-controller.md) (Next project)
