# 149 - IFTTT Motion breach alert (Intruder detection -> trigger phone SMS)

Build a cloud-connected security system on the ESP32 that monitors a PIR motion sensor on GPIO 14. When motion is detected while armed, it triggers a local buzzer alarm, flashes an alert LED, and sends an IFTTT Webhook request to trigger a SMS notification on your phone.

## Goal
Learn how to read digital PIR motion sensors, manage armed/disarmed security states, compile HTTP Webhook requests, and host interactive local web servers to control the system.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A PIR sensor is on GPIO 14, an LED on GPIO 12, and a buzzer on GPIO 13. A local webpage lets you Arm or Disarm the security system. When Armed, detecting motion triggers the buzzer, flashes the LED, and sends an HTTP POST request to IFTTT (`maker.ifttt.com/trigger/motion_breach/with/key/[api_key]`). A latch prevents duplicate notifications, resetting after 30 seconds of inactivity.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT (Digital) | GPIO14 | Yellow | Motion sensor input |
| LED | Anode (+) | GPIO12 | Red | Alarm indicator output |
| Buzzer | Positive (+) | GPIO13 | Purple | Alarm siren device |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED anode. Power the PIR sensor from the 5V Vin rail.

## Code
```cpp
// IFTTT Motion breach alert (PIR sensor + IFTTT Webhook + Arm/Disarm Console)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// IFTTT Webhooks configuration
const String iftttEvent = "motion_breach";
const String iftttKey = "ifttt_key_XYZ123456789"; // Place your IFTTT Webhooks Maker key here

const int PIR_PIN = 14;
const int LED_PIN = 12;
const int BUZZER_PIN = 13;

WebServer server(80);

// Global state variables
bool systemArmed = false;
bool motionDetected = false;
bool alarmActive = false;
bool webhookLatched = false;

unsigned long motionStartTime = 0;
const unsigned long ALARM_TIMEOUT_MS = 30000; // Keep alarm active for 30 seconds

String iftttSyncStatus = "Idle";

// Trigger IFTTT Webhook via HTTP POST
void triggerIFTTTWebhook() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Endpoint format: http://maker.ifttt.com/trigger/[event]/with/key/[api_key]
    String url = "http://maker.ifttt.com/trigger/" + iftttEvent + "/with/key/" + iftttKey;
    
    http.begin(url);
    http.addHeader("Content-Type", "application/json");
    
    // Value1: Status update details
    String jsonPayload = "{\"value1\":\"Security Breach - Intruder Detected!\"}";
    
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
  String json = "{\"armed\":" + String(systemArmed ? 1 : 0) + 
                 ",\"motion\":" + String(motionDetected ? 1 : 0) + 
                 ",\"alarm\":" + String(alarmActive ? 1 : 0) + 
                 ",\"sync\":\"" + iftttSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to change system armed state
void handlePostMode() {
  if (server.hasArg("mode")) {
    String mode = server.arg("mode");
    if (mode == "arm") {
      systemArmed = true;
      Serial.println("[Security] System ARMED.");
    } else if (mode == "disarm") {
      systemArmed = false;
      alarmActive = false;
      webhookLatched = false;
      digitalWrite(LED_PIN, LOW);
      noTone(BUZZER_PIN);
      Serial.println("[Security] System DISARMED.");
    }
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Home Security Panel</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-indicator { font-size: 32px; font-weight: bold; margin: 20px 0; color: #64748b; text-transform: uppercase; }\n";
  html += "  .status-indicator.armed { color: #f59e0b; }\n";
  html += "  .status-indicator.breach { color: #ef4444; animation: flash 1s infinite; }\n";
  html += "  .status-indicator.disarmed { color: #10b981; }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 25px; }\n";
  html += "  .btn { flex-grow: 1; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.arm-btn { background-color: #92400e; border-color: #b45309; }\n";
  html += "  .btn.arm-btn:hover { background-color: #b45309; }\n";
  html += "  .btn.disarm-btn { background-color: #065f46; border-color: #047857; }\n";
  html += "  .btn.disarm-btn:hover { background-color: #047857; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "  @keyframes flash { 50% { opacity: 0.5; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Intrusion Alarm Hub</h1>\n";
  html += "  <div id=\"systemMode\" class=\"status-indicator disarmed\">Disarmed</div>\n";
  
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn arm-btn\" onclick=\"setMode('arm')\">ARM SYSTEM</button>\n";
  html += "    <button class=\"btn disarm-btn\" onclick=\"setMode('disarm')\">DISARM</button>\n";
  html += "  </div>\n";
  
  html += "  <h3>IFTTT Integration</h3>\n";
  html += "  <div id=\"syncStatus\" style=\"font-size: 13px; font-family: monospace; color: #94a3b8; background-color:#0f172a; padding:10px; border-radius:6px; border:1px solid #334155;\">Idle</div>\n";
  
  html += "  <p class=\"footer\">IFTTT Event: motion_breach | Alarm Duration: 30s</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateSecurityHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const modeEl = document.getElementById('systemMode');\n";
  
  html += "        if (data.alarm === 1) {\n";
  html += "          modeEl.innerText = 'BREACH DETECTED';\n";
  html += "          modeEl.className = 'status-indicator breach';\n";
  html += "        } else if (data.armed === 1) {\n";
  html += "          modeEl.innerText = 'ARMED - SAFE';\n";
  html += "          modeEl.className = 'status-indicator armed';\n";
  html += "        } else {\n";
  html += "          modeEl.innerText = 'DISARMED';\n";
  html += "          modeEl.className = 'status-indicator disarmed';\n";
  html += "        }\n";
  
  html += "        document.getElementById('syncStatus').innerText = data.sync;\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function setMode(modeVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('mode', modeVal);\n";
  html += "    fetch('/api/mode', { method: 'POST', body: body })\n";
  html += "      .then(() => updateSecurityHUD());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateSecurityHUD();\n";
  html += "    setInterval(updateSecurityHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 Home Security starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/mode", HTTP_POST, handlePostMode);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 1. Read motion sensor state
  motionDetected = (digitalRead(PIR_PIN) == HIGH);
  
  // 2. Alarm Trigger Logic
  if (systemArmed) {
    if (motionDetected) {
      motionStartTime = millis(); // Refresh timeout timer while motion is actively detected
      
      if (!alarmActive) {
        alarmActive = true;
        Serial.println("[Security] MOTION DETECTED! Intruder alert activated.");
      }
      
      // Trigger cloud webhook (once per alarm cycle)
      if (!webhookLatched) {
        webhookLatched = true;
        triggerIFTTTWebhook();
      }
    }
    
    // Process alarm outputs (Blink LED and beep buzzer)
    if (alarmActive) {
      // Toggle LED state every 200 ms to flash
      static unsigned long lastFlash = 0;
      if (millis() - lastFlash >= 200) {
        lastFlash = millis();
        digitalWrite(LED_PIN, !digitalRead(LED_PIN));
        
        // Output alternating alarm pitch
        static bool toneToggle = false;
        toneToggle = !toneToggle;
        tone(BUZZER_PIN, toneToggle ? 1000 : 800);
      }
      
      // Check alarm timeout (turn off after 30 seconds of quiet)
      if (millis() - motionStartTime >= ALARM_TIMEOUT_MS) {
        alarmActive = false;
        webhookLatched = false; // Reset webhook latch
        digitalWrite(LED_PIN, LOW);
        noTone(BUZZER_PIN);
        Serial.println("[Security] Alarm timed out. Resetting system.");
      }
    }
  } else {
    // If system is disarmed, ensure outputs are shut down
    digitalWrite(LED_PIN, LOW);
    noTone(BUZZER_PIN);
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **PIR Motion Sensor**, **LED**, and **Buzzer** onto the canvas.
2. Wire PIR Sensor to **GPIO14**, LED to **GPIO12**, and Buzzer to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click "ARM SYSTEM". The web page indicator changes to orange `ARMED - SAFE`.
6. Trigger the simulated PIR sensor (click the widget and toggle motion to high).
7. Verify that the buzzer sounds, the LED flashes, and the webpage status flashes red `BREACH DETECTED`.
8. Click "DISARM" to silence the alarm.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Security] System ARMED.
[Security] MOTION DETECTED! Intruder alert activated.
[IFTTT] Triggering webhook URL: http://maker.ifttt.com/trigger/motion_breach/with/key/ifttt_key_XYZ123456789
[IFTTT] Webhook triggered successfully. Code: 200
```

## Expected Canvas Behavior
* Triggering the armed PIR sensor flashes the LED widget and plays alternating tones on the buzzer widget.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `systemArmed` | Mode flag that blocks alarm triggers when set to false (disarmed). |
| `digitalRead(PIR_PIN)` | Reads the digital output of the motion sensor. |
| `triggerIFTTTWebhook()` | Fires the HTTP request to trigger the SMS event. |
| `millis() - motionStartTime` | Checks if the 30-second quiet timeout has elapsed to clear the alarm. |

## Hardware & Safety Concept: Sensor Stabilization and Siren Power Isolations
* **PIR Stabilization**: When first powered on, PIR sensors require 30 to 60 seconds to calibrate their internal infrared reference grid. During this warm-up period, the sensor may output random false triggers. Make sure to implement a startup delay (e.g. ignoring PIR reads for the first 30 seconds of uptime) to prevent false alarms on boot.
* **Auto-arm Schedule**: Implement a fallback scheduler (Project 113) that automatically arms the security alarm at night (e.g. 10:00 PM) even if the user forgets to click Arm on the webpage dashboard.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a lock icon showing the armed/disarmed status.
2. **Keypad Arming**: Add a 4x4 Keypad (Project 114) to allow arming and disarming the security system using a physical PIN code.
3. **SPIFFS Breach Logger**: Log all breach times to a CSV file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| System triggers alarm immediately on arming | PIR sensor warm-up | Wait 60 seconds after powering on for the PIR sensor to stabilize |
| Webhook returns code 403 | Invalid IFTTT key | Double check your IFTTT Maker Key in the code |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [103 - ESP32 Smart Doorbell Alert with Web Notification](103-esp32-smart-doorbell-alert-iot-pir-chime-lcd-web-notification.md)
- [114 - ESP32 Keypad 4x4 Password Lock with Access Logs](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md)
- [150 - Dual-protocol WiFi-to-SD logger](150-esp32-dual-protocol-wifi-to-sd-logger.md) (Next project)
