# 148 - IFTTT Alarm Notification node (Gas leak -> trigger email alert)

Build an environmental safety alarm node on the ESP32 that samples an MQ-2 gas sensor on GPIO 34, triggers a local buzzer alarm on GPIO 12, closes a exhaust fan relay on GPIO 13, and sends an IFTTT Webhook request to trigger an email alert when gas concentration exceeds safe limits.

## Goal
Learn how to implement threshold checks with hysteresis, trigger secure cloud webhooks (IFTTT Webhooks Maker channel), compile JSON event payloads, and construct local alarm overrides.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MQ-2 gas sensor is on GPIO 34, a buzzer on GPIO 12, and a relay on GPIO 13. If gas levels exceed a threshold (e.g. 40%), the ESP32 sounds the buzzer, closes the relay, and sends an HTTP POST request to IFTTT (`maker.ifttt.com/trigger/gas_leak/with/key/[api_key]`) to email the user. The alarm latches to prevent multiple emails for the same event, resetting only when gas levels drop to safe levels.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the MQ-2 gas sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | AO (Analog Out) | GPIO34 | Yellow | Gas concentration input |
| Buzzer | Positive (+) | GPIO12 | Purple | Alarm audio device |
| Relay Module | IN (Signal) | GPIO13 | Blue | Exhaust fan relay |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the MQ-2 and the relay module from the 5V Vin rail.

## Code
```cpp
// IFTTT Alarm Notification node (MQ-2 sensor + IFTTT Webhook client + Alarm)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// IFTTT Webhooks configuration
const String iftttEvent = "gas_leak";
const String iftttKey = "ifttt_key_XYZ123456789"; // Place your IFTTT Webhooks Maker key here

const int MQ2_PIN = 34;
const int BUZZER_PIN = 12;
const int RELAY_PIN = 13;

WebServer server(80);

// Global values
int rawGas = 0;
float gasPct = 0.0;
bool alarmActive = false;
bool webhookLatched = false;

String iftttSyncStatus = "Idle";

// Trigger IFTTT Webhook via HTTP POST
void triggerIFTTTWebhook(float value) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Endpoint format: http://maker.ifttt.com/trigger/[event]/with/key/[api_key]
    String url = "http://maker.ifttt.com/trigger/" + iftttEvent + "/with/key/" + iftttKey;
    
    http.begin(url);
    http.addHeader("Content-Type", "application/json");
    
    // Pass gas level as value1 in the JSON payload
    String jsonPayload = "{\"value1\":\"" + String(value, 1) + "%\"}";
    
    Serial.printf("[IFTTT] Triggering webhook URL: %s\n", url.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode > 0) {
      String response = http.getString();
      iftttSyncStatus = "Triggered (Response: " + response + ")";
      Serial.printf("[IFTTT] Webhook triggered successfully. Code: %d\n", httpResponseCode);
    } else {
      iftttSyncStatus = "Error: " + String(httpResponseCode);
      Serial.printf("[IFTTT] Error sending POST webhook: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    iftttSyncStatus = "WiFi Offline";
    Serial.println("[IFTTT] Sync blocked. WiFi offline.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"gas\":" + String(gasPct, 1) + 
                 ",\"alarm\":" + String(alarmActive ? 1 : 0) + 
                 ",\"sync\":\"" + iftttSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Safety Alarm HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .gas-display { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .gas-display.danger { color: #ef4444; animation: blink 1s infinite; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.danger { background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "  @keyframes blink { 50% { opacity: 0.5; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Gas Safety Monitor</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"badge\">System OK</div>\n";
  html += "  <div id=\"gasVal\" class=\"gas-display\">0.0 %</div>\n";
  
  html += "  <h3>Cloud Notification Log</h3>\n";
  html += "  <div id=\"syncStatus\" style=\"font-size: 14px; font-family: monospace; color: #94a3b8; background-color:#0f172a; padding:10px; border-radius:6px; border:1px solid #334155;\">Idle</div>\n";
  
  html += "  <p class=\"footer\">IFTTT Event: gas_leak | Threshold: 40.0%</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateSafetyHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const valEl = document.getElementById('gasVal');\n";
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  
  html += "        valEl.innerText = data.gas.toFixed(1) + ' %';\n";
  
  html += "        if(data.alarm === 1) {\n";
  html += "          valEl.className = 'gas-display danger';\n";
  html += "          bdgEl.innerText = 'GAS LEAK DETECTED';\n";
  html += "          bdgEl.className = 'badge danger';\n";
  html += "        } else {\n";
  html += "          valEl.className = 'gas-display';\n";
  html += "          bdgEl.innerText = 'System OK';\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  
  html += "        document.getElementById('syncStatus').innerText = data.sync;\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateSafetyHUD();\n";
  html += "    setInterval(updateSafetyHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(MQ2_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
  
  Serial.println("\nESP32 Alarm Notification starting...");
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
  
  // 1. Read Gas sensor value
  rawGas = analogRead(MQ2_PIN);
  gasPct = ((float)rawGas / 4095.0) * 100.0;
  
  // 2. Threshold Check with Hysteresis Deadband
  // Threshold to trigger: 40%. Safe level to clear: 30%.
  if (gasPct > 40.0) {
    alarmActive = true;
    
    // Actuate exhaust relay and sound buzzer
    digitalWrite(RELAY_PIN, HIGH);
    tone(BUZZER_PIN, 1500);
    
    // Trigger cloud notification once (latch)
    if (!webhookLatched) {
      webhookLatched = true;
      triggerIFTTTWebhook(gasPct);
    }
  } 
  else if (gasPct < 30.0) {
    alarmActive = false;
    webhookLatched = false; // Reset latch when environment returns to safe levels
    
    // Turn off outputs
    digitalWrite(RELAY_PIN, LOW);
    noTone(BUZZER_PIN);
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate MQ-2), **Buzzer**, and **Relay** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Buzzer to **GPIO12**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, keep the potentiometer slider low. Verify that the system shows "System OK".
6. Turn the potentiometer slider above 50% (gas leak simulated). Verify that the buzzer sounds, the relay closes, the status page flashes RED, and the console logs the IFTTT webhook request.
7. Return the slider below 20%. Verify that the alarm stops.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[IFTTT] Triggering webhook URL: http://maker.ifttt.com/trigger/gas_leak/with/key/ifttt_key_XYZ123456789
[IFTTT] Webhook triggered successfully. Code: 200
```

## Expected Canvas Behavior
* Sliding the simulated gas slider above 40% triggers the buzzer widget, closes the relay widget, and initiates the IFTTT webhook request.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `gasPct > 40.0` | Threshold comparison checking if the gas level is dangerous. |
| `!webhookLatched` | Latch flag check that prevents sending multiple emails for a single leak event. |
| `triggerIFTTTWebhook(...)` | Packages the event data and executes the HTTP POST request to IFTTT. |
| `gasPct < 30.0` | Hysteresis lower limit that clears the alarm when gas levels drop. |

## Hardware & Safety Concept: Hysteresis Bands and Webhook Rate Protection
* **Hysteresis Bands**: Environmental sensors fluctuate due to noise (e.g. reading 39.9% then 40.1% repeatedly). If the threshold was a single value (40%), the alarm would turn ON and OFF rapidly. Implementing a **hysteresis band** (turn ON at 40%, turn OFF at 30%) ensures clean state transitions and protects relays from wearing out.
* **Webhook Rate Protection**: Webhook endpoints can block devices that send too many requests. Using a **latch flag** (`webhookLatched`) ensures that a gas leak triggers exactly one webhook alert when the threshold is crossed, instead of sending requests continuously in the loop.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning biohazard icon during an alarm.
2. **Exhaust fan speed control**: Use ESP32 PWM (LEDC peripheral) to run the exhaust fan relay at full speed during an alarm, and low speed for ventilation.
3. **SPIFFS Event Logger**: Save the timestamp of every gas leak alarm event to SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| IFTTT Webhook returns 401 Unauthorized | Invalid API key | Verify that you copied the correct API Key from the IFTTT Webhooks documentation page |
| Webhook triggers continuously | Latch flag failed | Check that the `webhookLatched` flag is set to true immediately after triggering the webhook |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [107 - ESP32 Fire Warning Station with Web Telemetry](107-esp32-fire-warning-station-iot-flame-buzzer-oled-web-telemetry.md)
- [147 - ESP32 Blynk Multi-relay switch board](147-esp32-blynk-multi-relay-switch-board.md)
- [149 - IFTTT Motion breach alert](149-esp32-ifttt-motion-breach-alert-intruder-detection-trigger-phone-sms.md) (Next project)
