# 190 - NTP Calendar Clock with Web Scheduler

Build a calendar-based event scheduling controller on the ESP32 that synchronizes local system time with NTP servers, hosts an administrative web dashboard to set custom daily ON/OFF trigger times, and controls a physical relay on GPIO 14.

## Goal
Learn how to configure Network Time Protocol (NTP) daemons, process local timezone configurations, build interactive scheduler interfaces with time inputs, and verify calendar events.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It synchronizes its internal clock with public NTP servers using a specified time zone offset. An HTTP server runs on port 80. Navigating to the ESP32's IP address displays a scheduling page showing the current synced time and date. The user enters a Start Time (e.g., `08:00`) and an End Time (e.g., `18:00`). The ESP32 automatically triggers a relay on GPIO 14 and an LED on GPIO 12 during the scheduled window.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO14 | Yellow | Controls heavy AC load |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 220 Ω current-limiting resistor in series with the LED.

## Code
```cpp
// NTP Calendar Clock (NTP Sync + Timezone offset + Preferences NVS + Web scheduler + Relay controller)
#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <time.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NTP Configuration
const char* ntpServer = "pool.ntp.org";
const long gmtOffset_sec = 19800; // Timezone offset in seconds (e.g. IST = +5:30 = 19800)
const int daylightOffset_sec = 0;

const int RELAY_PIN = 14;
const int LED_PIN = 12;

WebServer server(80);
Preferences preferences;

// Scheduler state variables
bool schedulerEnabled = false;
int startHour = 8;
int startMinute = 0;
int endHour = 18;
int endMinute = 0;

bool relayState = false;

// Get formatted local time string
String getFormattedTime() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    return "Time Sync Error";
  }
  char timeStringBuff[50];
  strftime(timeStringBuff, sizeof(timeStringBuff), "%A, %B %d %Y %H:%M:%S", &timeinfo);
  return String(timeStringBuff);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>NTP Web Scheduler</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .time-display { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: center; font-size: 16px; color: #10b981; font-weight: bold; margin-bottom: 25px; }\n";
  html += "  .form-group { margin-bottom: 20px; }\n";
  html += "  label { display: block; font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; margin-bottom: 8px; }\n";
  html += "  input[type=time] { width: 100%; padding: 12px; border-radius: 6px; border: 1px solid #334155; background-color: #0f172a; color: #f8fafc; font-size: 16px; box-sizing: border-box; }\n";
  html += "  .checkbox-group { display: flex; align-items: center; gap: 10px; margin: 25px 0; }\n";
  html += "  input[type=checkbox] { width: 20px; height: 20px; cursor: pointer; }\n";
  html += "  .btn-submit { background-color: #38bdf8; color: #0f172a; width: 100%; padding: 14px; font-size: 15px; font-weight: bold; border-radius: 6px; cursor: pointer; border: none; transition: background-color 0.2s; }\n";
  html += "  .btn-submit:hover { background-color: #0ea5e9; }\n";
  html += "  .status-badge { text-align: center; margin-top: 25px; font-size: 14px; font-weight: bold; color: #94a3b8; }\n";
  html += "  .status-indicator { display: inline-block; width: 12px; height: 12px; border-radius: 50%; margin-right: 8px; }\n";
  html += "  .status-on { background-color: #10b981; }\n";
  html += "  .status-off { background-color: #ef4444; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>NTP System Scheduler</h1>\n";
  
  html += "  <div class=\"time-display\" id=\"clock\">" + getFormattedTime() + "</div>\n";
  
  // Settings Form
  html += "  <form method=\"POST\" action=\"/save\">\n";
  
  // Format times for display in input
  char startTimeBuf[10];
  char endTimeBuf[10];
  sprintf(startTimeBuf, "%02d:%02d", startHour, startMinute);
  sprintf(endTimeBuf, "%02d:%02d", endHour, endMinute);
  
  html += "    <div class=\"form-group\">\n";
  html += "      <label>Start Trigger Time</label>\n";
  html += "      <input type=\"time\" name=\"startTime\" value=\"" + String(startTimeBuf) + "\">\n";
  html += "    </div>\n";
  
  html += "    <div class=\"form-group\">\n";
  html += "      <label>End Trigger Time</label>\n";
  html += "      <input type=\"time\" name=\"endTime\" value=\"" + String(endTimeBuf) + "\">\n";
  html += "    </div>\n";
  
  html += "    <div class=\"checkbox-group\">\n";
  html += "      <input type=\"checkbox\" name=\"enable\" id=\"enableCheck\" " + String(schedulerEnabled ? "checked" : "") + ">\n";
  html += "      <label for=\"enableCheck\" style=\"margin-bottom:0;cursor:pointer;\">Enable Auto-Scheduler</label>\n";
  html += "    </div>\n";
  
  html += "    <button type=\"submit\" class=\"btn-submit\">Save Settings</button>\n";
  html += "  </form>\n";
  
  // Current Relay State Output
  html += "  <div class=\"status-badge\">\n";
  html += "    <span class=\"status-indicator " + String(relayState ? "status-on" : "status-off") + "\"></span>\n";
  html += "    Relay State: " + String(relayState ? "ON" : "OFF");
  html += "  </div>\n";
  
  html += "</div>\n";
  
  // Auto-refresh Clock Script
  html += "<script>\n";
  html += "  setInterval(function() {\n";
  html += "    fetch('/time').then(response => response.text()).then(text => {\n";
  html += "      document.getElementById('clock').innerText = text;\n";
  html += "    });\n";
  html += "  }, 1000);\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST route returning updated raw time string
void handleGetTime() {
  server.send(200, "text/plain", getFormattedTime());
}

// POST endpoint saving settings to Preferences NVS
void handleSave() {
  if (server.hasArg("startTime") && server.hasArg("endTime")) {
    String sTime = server.arg("startTime");
    String eTime = server.arg("endTime");
    
    startHour = sTime.substring(0, 2).toInt();
    startMinute = sTime.substring(3, 5).toInt();
    endHour = eTime.substring(0, 2).toInt();
    endMinute = eTime.substring(3, 5).toInt();
    
    schedulerEnabled = server.hasArg("enable");
    
    // Save settings permanently in NVS partition
    preferences.begin("scheduler", false);
    preferences.putBool("enabled", schedulerEnabled);
    preferences.putInt("startH", startHour);
    preferences.putInt("startM", startMinute);
    preferences.putInt("endH", endHour);
    preferences.putInt("endM", endMinute);
    preferences.end();
    
    Serial.println("[Scheduler] Settings updated and saved to NVS.");
    
    // Redirect back to root page
    server.sendHeader("Location", "/", true);
    server.send(303);
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Evaluate time matching conditions to toggle outputs
void checkScheduler() {
  struct tm timeinfo;
  if (!getLocalTime(&timeinfo)) {
    return;
  }
  
  int currentHour = timeinfo.tm_hour;
  int currentMinute = timeinfo.tm_min;
  
  if (schedulerEnabled) {
    // Convert times to total minutes from midnight for easy comparison
    int currentTotalMinutes = currentHour * 60 + currentMinute;
    int startTotalMinutes = startHour * 60 + startMinute;
    int endTotalMinutes = endHour * 60 + endMinute;
    
    bool shouldBeOn = false;
    
    if (startTotalMinutes < endTotalMinutes) {
      // Normal schedule (e.g. 08:00 to 18:00)
      shouldBeOn = (currentTotalMinutes >= startTotalMinutes && currentTotalMinutes < endTotalMinutes);
    } else {
      // Overnight schedule (e.g. 20:00 to 06:00)
      shouldBeOn = (currentTotalMinutes >= startTotalMinutes || currentTotalMinutes < endTotalMinutes);
    }
    
    if (shouldBeOn != relayState) {
      relayState = shouldBeOn;
      digitalWrite(RELAY_PIN, relayState ? HIGH : LOW);
      digitalWrite(LED_PIN, relayState ? HIGH : LOW);
      Serial.printf("[Scheduler] Relay state altered to: %s\n", relayState ? "ON" : "OFF");
    }
  } else {
    // If scheduler disabled, force relay OFF
    if (relayState) {
      relayState = false;
      digitalWrite(RELAY_PIN, LOW);
      digitalWrite(LED_PIN, LOW);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(LED_PIN, LOW);
  
  // 1. Initialize Preferences NVS namespace and load settings
  preferences.begin("scheduler", true); // Open read-only mode first
  schedulerEnabled = preferences.getBool("enabled", false);
  startHour = preferences.getInt("startH", 8);
  startMinute = preferences.getInt("startM", 0);
  endHour = preferences.getInt("endH", 18);
  endMinute = preferences.getInt("endM", 0);
  preferences.end();
  
  Serial.println("\nESP32 Calendar Scheduler node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // 2. Synchronize clock with NTP server
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  Serial.println("Syncing NTP Time...");
  
  struct tm timeinfo;
  while (!getLocalTime(&timeinfo)) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nTime synced successfully.");
  Serial.print("Current Time: ");
  Serial.println(getFormattedTime());
  
  // 3. Web Server routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/time", HTTP_GET, handleGetTime);
  server.on("/save", HTTP_POST, handleSave);
  server.begin();
  
  Serial.println("HTTP Server active.");
}

void loop() {
  server.handleClient();
  
  // Periodically evaluate scheduler (every 5 seconds)
  static unsigned long lastCheck = 0;
  if (millis() - lastCheck > 5000) {
    lastCheck = millis();
    checkScheduler();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, and **LED** onto the canvas.
2. Wire Relay input to **GPIO14** and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set the Start Time to 5 minutes before the current time shown on the webpage, and the End Time to 5 minutes after the current time. Check "Enable Auto-Scheduler" and click "Save Settings".
6. Verify that the relay widget on the canvas turns ON (changes color) and the LED turns ON.
7. Set the start and end times outside the current window. Click Save and verify that the relay and LED turn OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Syncing NTP Time...
Time synced successfully.
Current Time: Saturday, July 11 2026 04:32:00
HTTP Server active.
[Scheduler] Settings updated and saved to NVS.
[Scheduler] Relay state altered to: ON
```

## Expected Canvas Behavior
* Booting the system fetches NTP timezone offsets. Modifying settings via the web interface triggers the relay and LED widgets.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `configTime(...)` | Configures timezone values and begins automatic NTP synchronization. |
| `getLocalTime(&timeinfo)` | Populates the tm struct containing calendar hour, minute, second, day, month, and year. |
| `preferences.begin("scheduler", ...)` | Mounts the non-volatile storage (NVS) sector to save config parameters across boot cycles. |

## Hardware & Safety Concept: NTP Time Drift and Overnight Transitions
* **NTP Time Drift**: The ESP32's internal crystal oscillator drifts over time (up to 4–5 seconds per day). To keep the clock accurate, the ESP32 periodically updates its internal time in the background from NTP servers. If the internet connection drops, the internal calendar continues to run on CPU timer ticks (keeping the schedule working offline).
* **Overnight Transitions**: Schedulers must handle wrap-around cases where the start time is later than the end time (e.g. start at 22:00, end at 06:00). This requires using logic that checks if the current time is after the start time OR before the end time, rather than a simple AND condition.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current date, time, and relay status.
2. **Weekly Scheduler**: Expand the settings to select specific active days of the week (e.g. run only on Mondays and Wednesdays) using a bitmask.
3. **Manual Override**: Add a button on the webpage that temporarily overrides the schedule for 1 hour.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Synced time is wrong | Incorrect timezone offset | Change `gmtOffset_sec` to match your timezone. For example, EST is `-18000` (-5 hours), and GMT is `0` |
| Time fails to sync | UDP Port 123 blocked | NTP operates using UDP packets on port 123. Ensure your router does not block outgoing NTP requests |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [34 - NTP Time Server sync](../beginner/34-ntp-time-server-sync-display-time.md)
- [112 - DS3231 Real Time Clock Web Sync Display](../intermediate/112-ds3231-real-time-clock-web-sync-display.md)
- [191 - Autonomous Robot HUD IoT Node](191-autonomous-robot-hud-iot-node.md) (Next project)
