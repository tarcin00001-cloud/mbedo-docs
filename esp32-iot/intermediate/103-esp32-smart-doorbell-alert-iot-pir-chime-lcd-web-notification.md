# 103 - Smart Doorbell Alert IoT (PIR + Chime + LCD + Web notification)

Build a web-connected smart doorbell on the ESP32 that detects motion using a PIR sensor on GPIO 14, displays visitor notifications on a 16x2 I2C LCD, sounds a ding-dong chime on a passive buzzer (GPIO 15), and updates a web dashboard with real-time logs.

## Goal
Learn how to read digital PIR motion sensors, play two-tone chimes using the `tone()` API, manage non-blocking alert timeouts, drive I2C LCDs, and maintain rolling event logs on a web dashboard.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A PIR sensor is on GPIO 14, a buzzer on GPIO 15, and a 16x2 I2C LCD on I2C (GPIO 21/22). When a visitor is detected, the LCD displays "VISITOR DETECTED", the buzzer plays a ding-dong chime, and the event is logged. A web page shows the doorbell status and a list of recent visits.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| PIR Motion Sensor | `button` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a pushbutton configured as active-HIGH (external pull-down or code mapping) to simulate the PIR motion sensor output.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT (Signal) | GPIO14 | Yellow | Motion detection input |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Doorbell chime output |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the PIR, buzzer, and LCD from the 5V Vin rail.

## Code
```cpp
// Smart Doorbell Alert IoT (PIR Chime triggering + LCD readouts + Web notification log)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int PIR_PIN = 14;
const int BUZZER_PIN = 15;

WebServer server(80);

bool visitorActive = false;
unsigned long lastVisitorTime = 0;
const unsigned long VISITOR_TIMEOUT_MS = 5000; // Reset display after 5 seconds

// Rolling Event Log (Stores last 5 entries)
String visitLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addVisitLog(String message) {
  for (int i = 4; i > 0; i--) {
    visitLogs[i] = visitLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  visitLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[DOORBELL] " + visitLogs[0]);
}

// Play musical Ding-Dong chime
void soundDingDong() {
  Serial.println("[Audio] Playing Ding-Dong chime...");
  
  // Note 1: "Ding" (880 Hz - A5)
  tone(BUZZER_PIN, 880);
  delay(250);
  
  // Note 2: "Dong" (659 Hz - E5)
  tone(BUZZER_PIN, 659);
  delay(350);
  
  noTone(BUZZER_PIN);
}

// Update the 16x2 LCD status screen
void updateLCDDisplay() {
  lcd.clear();
  if (visitorActive) {
    lcd.setCursor(0, 0);
    lcd.print("VISITOR DETECTED");
    lcd.setCursor(0, 1);
    lcd.print("Chime Active");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Doorbell System");
    lcd.setCursor(0, 1);
    lcd.print("Status: Idle");
  }
}

// HTTP API endpoint returning JSON data
void handleGetDoorbellData() {
  String stateStr = visitorActive ? "VISITOR DETECTED" : "IDLE";
  String json = "{\"state\":\"" + stateStr + "\",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + visitLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Doorbell Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 16px; text-transform: uppercase; margin: 20px 0; }\n";
  html += "  .idle { background-color: #1e293b; color: #64748b; border: 1px solid #334155; }\n";
  html += "  .active { background-color: #065f46; color: #a7f3d0; animation: pulse 1s infinite; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  @keyframes pulse { 0%, 100% { transform: scale(1); } 50% { transform: scale(1.02); } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Doorbell Console</h1>\n";
  html += "  <div id=\"bellBadge\" class=\"status-badge idle\">IDLE</div>\n";
  
  html += "  <h3>Visitor History</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Activity Logs</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateDoorbellStatus() {\n";
  html += "    fetch('/api/doorbell')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('bellBadge');\n";
  html += "        bdgEl.innerText = data.state;\n";
  
  html += "        if (data.state === 'IDLE') {\n";
  html += "          bdgEl.className = 'status-badge idle';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge active';\n";
  html += "        }\n";
  
  html += "        const tableBody = document.getElementById('logTable');\n";
  html += "        tableBody.innerHTML = '';\n";
  html += "        data.logs.forEach(log => {\n";
  html += "          if (log.length > 0) {\n";
  html += "            tableBody.innerHTML += '<tr><td>' + log + '</td></tr>';\n";
  html += "          }\n";
  html += "        });\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateDoorbellStatus();\n";
  html += "    setInterval(updateDoorbellStatus, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(PIR_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  updateLCDDisplay();
  
  Serial.println("\nESP32 Smart Doorbell Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/doorbell", handleGetDoorbellData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addVisitLog("Doorbell Active");
}

void loop() {
  server.handleClient();
  
  // Read PIR sensor (HIGH when motion detected, LOW when clear)
  bool motionDetected = (digitalRead(PIR_PIN) == HIGH);
  unsigned long now = millis();
  
  if (motionDetected) {
    if (!visitorActive) {
      visitorActive = true;
      lastVisitorTime = now;
      
      // Update screen and sound chime
      updateLCDDisplay();
      soundDingDong();
      
      addVisitLog("Visitor Detected");
    } else {
      // Keep resetting the timeout if motion continues
      lastVisitorTime = now;
    }
  } else {
    // Clear warning status after timeout has elapsed
    if (visitorActive && (now - lastVisitorTime >= VISITOR_TIMEOUT_MS)) {
      visitorActive = false;
      updateLCDDisplay();
      Serial.println("[Doorbell] Status returned to Idle.");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Pushbutton** (configured as active-HIGH to simulate the PIR sensor), **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire Button to **GPIO14**, Buzzer to **GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the button widget. Verify that the buzzer plays the chime, the LCD shows "VISITOR DETECTED", and the web dashboard logs the visitor.
6. Wait 5 seconds without clicking the button. Verify that the system returns to "Idle".

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[DOORBELL] [0s] Doorbell Active
[Audio] Playing Ding-Dong chime...
[DOORBELL] [8s] Visitor Detected
[Doorbell] Status returned to Idle.
```

LCD Display:
```
VISITOR DETECTED
Chime Active
```

## Expected Canvas Behavior
* Pressing the simulated PIR button sounds the local buzzer widget (pulsing green) and displays visitor info on the LCD widget.
* The web history table logs the timestamped visit.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `motionDetected = (digitalRead(PIR_PIN) == HIGH)` | Reads the active-HIGH PIR motion signal. |
| `soundDingDong()` | Plays a musical tone pattern on the passive buzzer. |
| `now - lastVisitorTime >= VISITOR_TIMEOUT_MS` | Clears the active alarm state after 5 seconds of inactivity. |
| `updateLCDDisplay()` | Updates the LCD screen with the current system state. |

## Hardware & Safety Concept: PIR Sensor Warm-Up and Cooldown Delays
* **Warm-Up**: PIR sensors require a warm-up period (typically 30 to 60 seconds on hardware boot) to stabilize their pyroelectric elements, during which they may trigger false alarms. Your code should ignore triggers for the first 30 seconds of boot time.
* **Cooldown**: When a PIR sensor detects motion, it keeps its output pin HIGH for a hardware-configured period (adjustable via potentiometer on the module, usually 3 to 300 seconds). Software must use non-blocking timers to prevent the chime from re-triggering continuously during this period.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and draw a bell icon.
2. **Doorbell Button Override**: Add a pushbutton (GPIO 12) as a physical doorbell button that triggers the chime immediately.
3. **Mute switch**: Add a slide switch (GPIO 13) to mute the buzzer alarm while keeping the LCD and web notifications active.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the passive buzzer is connected to GPIO 15 |
| PIR triggers continuously | Warm-up noise | Ignore inputs during the first 30 seconds of startup |
| LCD does not print text | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [29 - ESP32 Wireless PIR motion alarm receiver](../beginner/29-esp32-wireless-pir-motion-alarm-receiver.md)
- [101 - ESP32 Automatic Barrier Gate IoT Console](101-esp32-automatic-barrier-gate-iot-console.md)
- [104 - ESP32 Night Light Controller IoT](104-esp32-night-light-controller-iot-ldr-pir-relay-web-override.md) (Next project)
