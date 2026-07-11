# 119 - OLED Digital Web Stop Watch

Build a high-precision digital stopwatch on the ESP32 that renders elapsed times (minutes, seconds, and centiseconds) on an SSD1306 OLED display on I2C (GPIO 21/22), processes local reset triggers via a button on GPIO 14, and exposes a web control panel to Start, Stop, and Reset the timer.

## Goal
Learn how to implement millisecond stopwatch accumulators, print formatted data to I2C OLED displays, serve fast-polling web dashboards, and handle HTTP POST control endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An SSD1306 OLED is on I2C (GPIO 21/22), and a physical button on GPIO 14. The OLED shows the stopwatch time (MM:SS:CC). Navigating to the ESP32's IP address displays a webpage displaying the stopwatch. Clicking Start, Stop, or Reset on the webpage controls the timer immediately. Pressing the physical button resets the timer to zero.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| SSD1306 OLED Display (128x64 I2C) | `oled_i2c` | Yes | Yes |
| Pushbutton (Reset) | `button` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| OLED Display | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Reset Button | Pin 1 / Pin 2 | GPIO14 / GND | Green / Black | Manual reset button |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the OLED from the 3.3V pin. Wire the Reset button as active-LOW (internal pull-up).

## Code
```cpp
// OLED Digital Web Stop Watch (Precision Millisecond Timer + Web Controls)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RESET_BUTTON_PIN = 14;

WebServer server(80);

// Stopwatch variables
unsigned long totalElapsedMs = 0;
unsigned long startTimestampMs = 0;
bool isTimerRunning = false;

// Format milliseconds to MM:SS:CC (Centiseconds)
String getFormattedTime(unsigned long ms) {
  unsigned long totalSeconds = ms / 1000;
  unsigned long minutes = totalSeconds / 60;
  unsigned long seconds = totalSeconds % 60;
  unsigned long centiseconds = (ms % 1000) / 10;
  
  char buffer[12];
  sprintf(buffer, "%02lu:%02lu:%02lu", minutes, seconds, centiseconds);
  return String(buffer);
}

// Update OLED screen text
void updateOLEDDisplay() {
  display.clearDisplay();
  
  // Title header
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.print("DIGITAL STOPWATCH");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  // Dynamic status line
  display.setCursor(0, 15);
  display.print("Status: ");
  display.print(isTimerRunning ? "RUNNING" : "STOPPED");
  
  // Centered Time digits
  display.setTextSize(2);
  display.setCursor(16, 32);
  display.print(getFormattedTime(totalElapsedMs));
  
  display.display();
}

// HTTP API endpoint returning JSON data
void handleGetStopwatchData() {
  String json = "{\"time\":\"" + getFormattedTime(totalElapsedMs) + "\"" +
                 ",\"running\":" + String(isTimerRunning ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to process controls
void handlePostControl() {
  if (server.hasArg("cmd")) {
    String cmd = server.arg("cmd");
    if (cmd == "start") {
      if (!isTimerRunning) {
        isTimerRunning = true;
        startTimestampMs = millis() - totalElapsedMs;
        Serial.println("[Stopwatch] Started.");
      }
    } 
    else if (cmd == "stop") {
      if (isTimerRunning) {
        isTimerRunning = false;
        totalElapsedMs = millis() - startTimestampMs;
        Serial.println("[Stopwatch] Stopped.");
      }
    } 
    else if (cmd == "reset") {
      isTimerRunning = false;
      totalElapsedMs = 0;
      Serial.println("[Stopwatch] Reset to zero.");
    }
    
    updateOLEDDisplay();
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Stopwatch Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .time-display { font-size: 48px; font-weight: 800; font-family: monospace; color: #10b981; margin: 25px 0; letter-spacing: 2px; }\n";
  html += "  .btn-group { display: flex; gap: 10px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn-start { background-color: #065f46; border-color: #059669; }\n";
  html += "  .btn-stop { background-color: #991b1b; border-color: #dc2626; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Network Stopwatch</h1>\n";
  html += "  <div id=\"stopwatchDisplay\" class=\"time-display\">00:00:00</div>\n";
  
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn btn-start\" onclick=\"triggerCommand('start')\">Start</button>\n";
  html += "    <button class=\"btn btn-stop\" onclick=\"triggerCommand('stop')\">Stop</button>\n";
  html += "    <button class=\"btn\" onclick=\"triggerCommand('reset')\">Reset</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 100 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop (fast polling for stopwatch display update)
  html += "<script>\n";
  html += "  function updateStopwatch() {\n";
  html += "    fetch('/api/stopwatch')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('stopwatchDisplay').innerText = data.time;\n";
  const startBtn = document.querySelector('.btn-start');
  const stopBtn = document.querySelector('.btn-stop');
  html += "        if (data.running === 1) {\n";
  html += "          startBtn.style.opacity = '0.5';\n";
  html += "          stopBtn.style.opacity = '1';\n";
  html += "        } else {\n";
  html += "          startBtn.style.opacity = '1';\n";
  html += "          stopBtn.style.opacity = '0.5';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function triggerCommand(cmdVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('cmd', cmdVal);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body })\n";
  html += "      .then(() => updateStopwatch());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateStopwatch();\n";
  html += "    setInterval(updateStopwatch, 100); // Fast poll (10 times a second)\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RESET_BUTTON_PIN, INPUT_PULLUP);
  
  // Initialize SSD1306 OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(R"(SSD1306 allocation failed)");
    for (;;);
  }
  display.clearDisplay();
  display.display();
  
  Serial.println("\nESP32 Stopwatch Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/stopwatch", HTTP_GET, handleGetStopwatchData);
  server.on("/api/control", HTTP_POST, handlePostControl);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateOLEDDisplay();
}

void loop() {
  server.handleClient();
  
  // 1. Calculate active stopwatch time accumulation
  if (isTimerRunning) {
    totalElapsedMs = millis() - startTimestampMs;
  }
  
  // 2. Read physical reset button state (active-LOW)
  if (digitalRead(RESET_BUTTON_PIN) == LOW) {
    delay(50); // Debounce delay
    if (digitalRead(RESET_BUTTON_PIN) == LOW) {
      isTimerRunning = false;
      totalElapsedMs = 0;
      Serial.println("[Physical Button] Stopwatch reset.");
      updateOLEDDisplay();
    }
  }
  
  // 3. Refresh OLED screen display
  static unsigned long lastOledUpdate = 0;
  if (millis() - lastOledUpdate >= 50) { // Fast screen refresh
    updateOLEDDisplay();
    lastOledUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Pushbutton** (configured as active-LOW for Reset), and **SSD1306 OLED** onto the canvas.
2. Wire Button to **GPIO14** and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click the "Start" button on the webpage. Verify that the stopwatch starts running on both the webpage and the OLED screen.
6. Click the "Stop" button. Click the physical reset button widget. Verify that the timer resets to zero.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Stopwatch] Started.
[Stopwatch] Stopped.
[Physical Button] Stopwatch reset.
```

OLED Display:
```
DIGITAL STOPWATCH
Status: RUNNING
00:14:85
```

## Expected Canvas Behavior
* Pressing the simulated reset button immediately resets the stopwatch digits printed on the OLED widget.
* Clicking web control buttons starts/stops/resets the display.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `totalElapsedMs = millis() - startTimestampMs` | Calculates active elapsed milliseconds. |
| `getFormattedTime(...)` | Formats milliseconds to minutes, seconds, and centiseconds. |
| `display.begin(SSD1306_SWITCHCAPVCC, 0x3C)` | Mounts the I2C interface of the SSD1306 OLED display. |
| `server.arg("cmd")` | Extracts the command (start/stop/reset) from the web POST request. |

## Hardware & Safety Concept: Timer Precision and Non-blocking Execution
* **Precision Limits**: The ESP32's internal oscillator is subject to thermal drift. For microsecond-precision measurements (e.g. measuring industrial motor speeds), use the ESP32's **hardware timers** (via ESP32 Timer Interrupts) instead of relying on the CPU-dependent `millis()` function.
* **Non-blocking loops**: Stopwatches require fast screen updates. Keep logic light and avoid blocking code in the main loop to ensure the display updates smoothly.

## Try This! (Challenges)
1. **Lap Timer**: Add a button on the webpage to record lap split times.
2. **Speed Indicator**: Calculate speed based on a set track distance (e.g. distance / time = speed).
3. **Trigger Alarm**: Sound a beep on a buzzer (GPIO 15) when the stopwatch reaches 1 minute.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The OLED display remains blank | I2C address mismatch | Verify the I2C address in `display.begin()` matches your module (usually `0x3C`) |
| Centiseconds skip numbers | Poll rate mismatch | Fast stopwatches must poll the API frequently (e.g. 100 ms) to display smooth updates |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 OLED Digital Clock NTP synced](../beginner/36-esp32-oled-digital-clock-ntp-synced.md)
- [112 - ESP32 DS3231 Real Time Clock Web Sync Display](112-esp32-ds3231-real-time-clock-web-sync-display.md)
- [120 - ESP32 Bluetooth to WiFi Telemetry Gateway](120-esp32-bluetooth-to-wifi-telemetry-gateway.md) (Next project)
