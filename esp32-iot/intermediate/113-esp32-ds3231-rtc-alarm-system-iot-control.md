# 113 - DS3231 RTC Alarm System IoT Control

Build an automated scheduler on the ESP32 that reads a DS3231 RTC module on I2C (GPIO 21/22), compares time values against configured target alarms, activates a relay output on GPIO 13 and a warning buzzer on GPIO 15 when the alarm time matches, and hosts a web dashboard to adjust schedule details.

## Goal
Learn how to read DS3231 RTC modules using RTClib, compare time structures, implement non-blocking alarm trigger windows, actuate multiple warning outputs, serve web pages, and handle HTTP POST variables.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DS3231 RTC is on I2C (GPIO 21/22), a relay on GPIO 13, and a buzzer on GPIO 15. The ESP32 compares the RTC time with the configured alarm time (default 08:00). When they match, the relay closes and the buzzer sounds for 10 seconds. Navigating to the ESP32's IP address displays the current time and allows setting the alarm hour and minute.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS3231 RTC Module | `i2c_device` | No | Yes |
| Relay Module | `relay` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, since the DS3231 is not physically on the canvas, the code uses simulated time offsets.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Relay Module | IN (Signal) | GPIO13 | Orange | Scheduled load relay |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Alarm sound output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the DS3231, relay, and buzzer from the 5V Vin rail.

## Code
```cpp
// DS3231 RTC Alarm System IoT Control (Scheduled Alarm Receiver Node)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <RTClib.h>

RTC_DS3231 rtc;
const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RELAY_PIN = 13;
const int BUZZER_PIN = 15;

WebServer server(80);

// Configured Alarm Variables (Default 08:00 AM)
int alarmHour = 8;
int alarmMinute = 0;
bool alarmEnabled = true;
bool alarmTripped = false;

// Format numbers as two digits
String formatTwoDigits(int number) {
  if (number < 10) return "0" + String(number);
  return String(number);
}

// HTTP API endpoint returning JSON data
void handleGetAlarmData() {
  DateTime now = rtc.now();
  String timeStr = formatTwoDigits(now.hour()) + ":" + 
                   formatTwoDigits(now.minute()) + ":" + 
                   formatTwoDigits(now.second());
  String alarmStr = formatTwoDigits(alarmHour) + ":" + 
                    formatTwoDigits(alarmMinute);
                    
  String json = "{\"time\":\"" + timeStr + "\"" +
                 ",\"alarm\":\"" + alarmStr + "\"" +
                 ",\"enabled\":" + String(alarmEnabled ? 1 : 0) +
                 ",\"tripped\":" + String(alarmTripped ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update alarm schedule
void handlePostConfigure() {
  if (server.hasArg("hour")) {
    alarmHour = server.arg("hour").toInt();
  }
  if (server.hasArg("minute")) {
    alarmMinute = server.arg("minute").toInt();
  }
  if (server.hasArg("enabled")) {
    alarmEnabled = (server.arg("enabled") == "1");
  }
  
  Serial.printf("[Scheduler] Alarm updated to %02d:%02d | Active: %s\n", 
                alarmHour, alarmMinute, alarmEnabled ? "YES" : "NO");
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Scheduler Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .time-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .alarm-val { font-size: 18px; color: #94a3b8; font-family: monospace; margin-bottom: 25px; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 13px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .enabled { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .disabled { background-color: #1e293b; color: #64748b; border: 1px solid #334155; }\n";
  html += "  .tripped { background-color: #991b1b; color: #fee2e2; animation: flash 1s infinite; }\n";
  html += "  .control-group { text-align: left; margin-top: 20px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 8px; }\n";
  html += "  .input-row { display: flex; gap: 10px; margin-bottom: 15px; }\n";
  html += "  input[type=number] { width: 100%; padding: 10px; border-radius: 6px; border: 1px solid #334155; background-color: #0f172a; color: white; font-size: 14px; text-align: center; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; font-size: 15px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  @keyframes flash { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Scheduler Terminal</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"badge disabled\">DISABLED</div>\n";
  html += "  <div id=\"timeDisplay\" class=\"time-val\">--:--:--</div>\n";
  html += "  <div id=\"alarmDisplay\" class=\"date-val\">Alarm Scheduled: --:--</div>\n";
  
  html += "  <div class=\"control-group\">\n";
  html += "    <label>Configure Alarm Time</label>\n";
  html += "    <div class=\"input-row\">\n";
  html += "      <input type=\"number\" id=\"hourInput\" min=\"0\" max=\"23\" placeholder=\"HH\">\n";
  html += "      <input type=\"number\" id=\"minuteInput\" min=\"0\" max=\"59\" placeholder=\"MM\">\n";
  html += "    </div>\n";
  html += "    <button class=\"btn\" onclick=\"saveAlarm()\">Update Alarm</button>\n";
  html += "  </div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateScheduler() {\n";
  html += "    fetch('/api/alarm')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('timeDisplay').innerText = data.time;\n";
  html += "        document.getElementById('alarmDisplay').innerText = 'Alarm Scheduled: ' + data.alarm;\n";
  
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        if (data.tripped === 1) {\n";
  html += "          bdgEl.innerText = 'ALARM ACTIVE';\n";
  html += "          bdgEl.className = 'badge tripped';\n";
  html += "        } else if (data.enabled === 1) {\n";
  html += "          bdgEl.innerText = 'ENABLED';\n";
  html += "          bdgEl.className = 'badge enabled';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'DISABLED';\n";
  html += "          bdgEl.className = 'badge disabled';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function saveAlarm() {\n";
  html += "    const hr = document.getElementById('hourInput').value;\n";
  html += "    const mn = document.getElementById('minuteInput').value;\n";
  
  html += "    if (hr !== '' && mn !== '') {\n";
  html += "      const body = new URLSearchParams();\n";
  html += "      body.append('hour', hr);\n";
  html += "      body.append('minute', mn);\n";
  html += "      body.append('enabled', '1');\n";
  
  html += "      fetch('/api/configure', { method: 'POST', body: body })\n";
  html += "        .then(() => {\n";
  html += "          updateScheduler();\n";
  html += "          document.getElementById('hourInput').value = '';\n";
  html += "          document.getElementById('minuteInput').value = '';\n";
  html += "        });\n";
  html += "    }\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateScheduler();\n";
  html += "    setInterval(updateScheduler, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Initialize DS3231 RTC
  if (!rtc.begin()) {
    Serial.println("[RTC Error] DS3231 module missing!");
  }
  
  Serial.println("\nESP32 Scheduled Alarm Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/alarm", handleGetAlarmData);
  server.on("/api/configure", HTTP_POST, handlePostConfigure);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 1. Read current time from local RTC
  DateTime now = rtc.now();
  
  // 2. Scheduled comparison logic
  // Trigger alarm during the first 10 seconds of the target minute
  if (alarmEnabled && (now.hour() == alarmHour) && (now.minute() == alarmMinute) && (now.second() < 10)) {
    if (!alarmTripped) {
      alarmTripped = true;
      Serial.println("[Trigger] Alarm conditions met. Actuating warning devices...");
    }
    
    digitalWrite(RELAY_PIN, HIGH); // Close relay
    
    // Pulse alarm tone on buzzer
    static unsigned long lastTone = 0;
    if (millis() - lastTone >= 200) {
      static bool soundOn = false;
      soundOn = !soundOn;
      if (soundOn) tone(BUZZER_PIN, 1200); else noTone(BUZZER_PIN);
      lastTone = millis();
    }
  } 
  else {
    if (alarmTripped) {
      alarmTripped = false;
      digitalWrite(RELAY_PIN, LOW); // Open relay
      noTone(BUZZER_PIN);          // Silence buzzer
      Serial.println("[Scheduler] Alarm window closed. Returning to monitoring state.");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, and **Buzzer** onto the canvas.
2. Wire Relay to **GPIO13** and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Look at the current time displayed on the webpage. Configure the alarm time to trigger 1 minute in the future.
6. Verify that when the time matches, the buzzer sounds, the relay activates, and the web page displays "ALARM ACTIVE" for 10 seconds.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Scheduler] Alarm updated to 01:28 | Active: YES
[Trigger] Alarm conditions met. Actuating warning devices...
[Scheduler] Alarm window closed. Returning to monitoring state.
```

## Expected Canvas Behavior
* When the RTC time reaches the target alarm time, the relay and buzzer widgets turn ON immediately.
* They stay active for 10 seconds before turning OFF.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `now.hour() == alarmHour` | Checks if the current hour matches the alarm hour. |
| `now.second() < 10` | Enforces a non-blocking 10-second alarm duration. |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contacts to trigger external loads. |
| `tone(BUZZER_PIN, 1200)` | Generates a pulsing alarm tone on the passive buzzer. |

## Hardware & Safety Concept: Scheduled Loads and Safety Thresholds
In scheduled automation systems (such as school bells or crop sprinklers):
1. **Safety Window**: Ensure that the trigger uses a specific duration (e.g. `now.second() < 10`) instead of matching just the minute (`now.minute() == alarmMinute`). If the second constraint is omitted, the alarm will stay active for the entire minute.
2. **Relay Safety**: When driving high-power AC loads, ensure the relay's contacts are rated for the load current to prevent them from melting.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a countdown timer until the next alarm.
2. **Double Schedule**: Add a second alarm schedule to trigger at a different time of day.
3. **Buzzer Snooze**: Add a physical button (GPIO 12) to snooze the alarm for 5 minutes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers for an entire minute | Second constraint missing | Verify that the `now.second() < 10` check is included in the condition |
| Relay does not switch | Relay signal pin miswired | Verify that the relay signal is connected to GPIO 13 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [112 - ESP32 DS3231 Real Time Clock Web Sync Display](112-esp32-ds3231-real-time-clock-web-sync-display.md)
- [114 - ESP32 Keypad 4x4 Password Lock IoT](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md) (Next project)
