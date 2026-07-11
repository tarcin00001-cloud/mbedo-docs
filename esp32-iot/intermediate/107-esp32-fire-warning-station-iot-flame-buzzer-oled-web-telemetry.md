# 107 - Fire Warning Station IoT (Flame + Buzzer + OLED + Web telemetry)

Build an integrated fire warning system on the ESP32 that samples an analog flame sensor on GPIO 34, displays status screens on an SSD1306 OLED display, triggers a pulsing audio alarm on a buzzer (GPIO 15) when fire is detected, and hosts a web dashboard displaying real-time safety telemetry.

## Goal
Learn how to sample analog infrared flame sensors, render warning graphics on I2C OLED displays, generate pulsing audio alarms, serve web pages, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A flame sensor is on GPIO 34, a buzzer on GPIO 15, and an SSD1306 OLED on I2C (GPIO 21/22). The OLED shows system safety readouts. Navigating to the ESP32's IP address displays a dashboard. If the flame sensor detects infrared heat (intensity > 50%), the buzzer sounds an alarm, the OLED flashes "FIRE WARNING!", and the web page displays a flashing hazard warning.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Flame Sensor | `potentiometer` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| SSD1306 OLED Display (128x64 I2C) | `oled_i2c` | Yes | Yes |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the flame sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flame Sensor | AO (Analog Out) | GPIO34 | Yellow | Infrared light level signal |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Warning buzzer |
| OLED Display | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the buzzer to GPIO 15. Power the OLED from the 3.3V rail and the flame sensor from the 5V Vin rail.

## Code
```cpp
// Fire Warning Station IoT (Flame sensor triggers + OLED alerts + Buzzer alarm + Web telemetry)
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

const int FLAME_PIN = 34;
const int BUZZER_PIN = 15;

WebServer server(80);

int flameIntensity = 0;
bool fireDetected = false;

// Return status string based on intensity
String getSafetyStatus() {
  return fireDetected ? "FIRE ALERT: HAZARD" : "SAFE";
}

// Update OLED screen diagnostics
void updateOLEDDisplay() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  
  // Title
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("FIRE MONITOR NODE");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  if (fireDetected) {
    // Flashing Alert box
    display.fillRect(0, 16, 128, 48, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(2);
    display.setCursor(10, 22);
    display.print("FIRE!");
    display.setTextSize(1);
    display.setCursor(10, 48);
    display.printf("Intensity: %d%%", flameIntensity);
  } else {
    display.setTextSize(1);
    display.setCursor(0, 20);
    display.print("Status: SECURED");
    display.setCursor(0, 36);
    display.printf("IR Reading: %d%%", flameIntensity);
    display.setCursor(0, 52);
    display.print("Web server active");
  }
  
  display.display();
}

// HTTP API endpoint returning JSON data
void handleGetFireData() {
  String json = "{\"flame\":" + String(flameIntensity) + 
                 ",\"status\":\"" + getSafetyStatus() + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Fire Safety Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .value { font-size: 64px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .value.alarm { color: #ef4444; animation: flash 0.5s infinite; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .badge.safe { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.alarm { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  @keyframes flash { 0%, 100% { opacity: 1; } 50% { opacity: 0.3; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Fire Safety Monitor</h1>\n";
  html += "  <div id=\"fireBadge\" class=\"badge safe\">SECURED</div>\n";
  html += "  <div id=\"fireDisplay\" class=\"value\">0%</div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateFireData() {\n";
  html += "    fetch('/api/fire')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('fireDisplay').innerText = data.flame + '%';\n";
  
  html += "        const valEl = document.getElementById('fireDisplay');\n";
  html += "        const bdgEl = document.getElementById('fireBadge');\n";
  
  html += "        bdgEl.innerText = data.status;\n";
  
  html += "        if (data.flame > 50) {\n";
  html += "          valEl.classList.add('alarm');\n";
  html += "          bdgEl.className = 'badge alarm';\n";
  html += "        } else {\n";
  html += "          valEl.classList.remove('alarm');\n";
  html += "          bdgEl.className = 'badge safe';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateFireData();\n";
  html += "    setInterval(updateFireData, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Initialize SSD1306 OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(R"(SSD1306 allocation failed)");
    for (;;);
  }
  display.clearDisplay();
  display.display();
  
  Serial.println("\nESP32 Fire Monitor Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/fire", handleGetFireData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateOLEDDisplay();
}

void loop() {
  server.handleClient();
  
  // Read analog flame sensor
  // Flame sensors output lower voltages in presence of infrared flame waves
  int rawVal = analogRead(FLAME_PIN);
  flameIntensity = map(rawVal, 4095, 0, 0, 100);
  if (flameIntensity < 0) flameIntensity = 0;
  if (flameIntensity > 100) flameIntensity = 100;
  
  fireDetected = (flameIntensity > 50);
  
  // Local alarm control loop
  if (fireDetected) {
    // Generate rapid pulsing buzzer alarm (siren sound)
    static unsigned long lastTone = 0;
    if (millis() - lastTone >= 100) { // Alternating state every 100 ms
      static bool soundOn = false;
      soundOn = !soundOn;
      if (soundOn) {
        tone(BUZZER_PIN, 2000);
      } else {
        noTone(BUZZER_PIN);
      }
      lastTone = millis();
    }
  } else {
    noTone(BUZZER_PIN); // Silence buzzer
  }
  
  // Refresh OLED screen periodically
  static unsigned long lastOledUpdate = 0;
  if (millis() - lastOledUpdate >= 250) {
    updateOLEDDisplay();
    lastOledUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the flame sensor), **Buzzer**, and **SSD1306 OLED** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Buzzer to **GPIO15**, and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer widget.
5. Move the potentiometer slider to simulate a fire (lower the value to simulate infrared light). Verify that the buzzer sounds its siren, the OLED screen flashes a warning, and the web page displays a flashing hazard alert.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/fire`):
```json
{"flame":75,"status":"FIRE ALERT: HAZARD"}
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates flame intensity levels.
* When the level exceeds 50%, the buzzer and OLED widgets actuate immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `map(rawVal, 4095, 0, 0, 100)` | Inverts and maps the flame sensor output (0V is fire, 3.3V is safe). |
| `flameIntensity > 50` | Warning comparison threshold trigger point. |
| `tone(BUZZER_PIN, 2000)` | Generates a high-frequency alarm tone on the passive buzzer. |
| `display.fillRect(...)` | Draws a filled rectangle on the OLED buffer to flash the screen. |

## Hardware & Safety Concept: Industrial Fire Detectors and Secondary Fail-Safes
* **Flame Detection**: Flame sensors use phototransistors designed to detect infrared light in the 760 nm to 1100 nm spectral range (which matches light emitted by fire). They are highly responsive but can trigger false alarms from direct sunlight or fluorescent lights.
* **Secondary Fail-Safes**: Always combine infrared flame sensors with secondary environmental sensors (like MQ-2 smoke detectors or NTC thermistors) to perform cross-validation before activating high-risk suppression relays (like water valves).

## Try This! (Challenges)
1. **OLED Temperature Log**: Add an NTC thermistor (Project 105) and display local temperatures alongside flame intensity.
2. **Suppressor Relay**: Add a relay (GPIO 13) to activate a water pump when the flame is active.
3. **SPIFFS logging**: Log alarm trigger statistics to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The OLED display remains blank | I2C address mismatch | Verify the I2C address in `display.begin()` matches your module (usually `0x3C`) |
| Alarm triggers in normal lighting | Sensor sensitivity too high | Adjust the sensor calibration potentiometer or increase the threshold value in the code |
| Webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 OLED Digital Clock NTP synced](../beginner/36-esp32-oled-digital-clock-ntp-synced.md)
- [82 - ESP32 MQTT Subscribe Buzzer melody player](82-esp32-mqtt-subscribe-buzzer-melody-player.md)
- [108 - ESP32 Intrusion Detector Alarm IoT](108-esp32-intrusion-detector-alarm-iot-laser-photoresistor-relay-web-notification.md) (Next project)
