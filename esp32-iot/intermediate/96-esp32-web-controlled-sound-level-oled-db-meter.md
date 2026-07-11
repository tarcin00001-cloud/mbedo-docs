# 96 - Web-controlled Sound Level OLED DB meter

Build an interactive sound monitoring station on the ESP32 that samples an analog sound sensor on GPIO 34, displays the decibel (dB) level on an SSD1306 OLED screen, hosts a web dashboard to set a warning threshold, and triggers a local alarm LED on GPIO 13 when the threshold is exceeded.

## Goal
Learn how to sample analog audio signals, map raw ADC readings to decibel approximations, render graphics on I2C OLED displays, serve interactive web control pages, and process HTTP POST configuration variables.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A sound sensor is on GPIO 34, an SSD1306 OLED on I2C (GPIO 21/22), and an LED on GPIO 13. The OLED displays the current sound level and threshold. Navigating to the ESP32's IP address opens a webpage showing the live sound level. The webpage has a slider to configure the decibel threshold. If the sound exceeds the threshold, the LED lights up immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog Sound Sensor | `potentiometer` | Yes | Yes |
| SSD1306 OLED Display (128x64 I2C) | `oled_i2c` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the sound sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Sound Sensor | OUT (Analog) | GPIO34 | Yellow | Audio envelope input |
| OLED Display | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Warning indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor. Power the OLED from the 3.3V rail and the sound sensor from the 5V Vin rail.

## Code
```cpp
// Web-controlled Sound Level OLED DB meter (Asynchronous Sound Alarm Node)
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

const int SOUND_PIN = 34;
const int LED_PIN = 13;

WebServer server(80);

// Alarm and calibration parameters
int soundThresholdDB = 75; // Default threshold set to 75 dB
int currentDB = 30;

// Convert raw analog reading to approximate decibels
int calculateDecibels(int rawAdc) {
  // Simple mapping for simulation: map raw ADC (0-4095) to dB range (30-100 dB)
  return map(rawAdc, 0, 4095, 30, 100);
}

void updateOLED() {
  display.clearDisplay();
  display.setTextColor(SSD1306_WHITE);
  
  // Title
  display.setTextSize(1);
  display.setCursor(0, 0);
  display.println("DB LEVEL MONITOR");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  // Decibel level
  display.setTextSize(3);
  display.setCursor(0, 18);
  display.print(currentDB);
  display.setTextSize(1);
  display.print(" dB");
  
  // Threshold limits
  display.setCursor(0, 48);
  display.printf("Threshold: %d dB", soundThresholdDB);
  
  // Alarm flag status
  display.setCursor(0, 56);
  if (currentDB > soundThresholdDB) {
    display.print("ALARM: OVER LIMIT!");
  } else {
    display.print("Status: Safe");
  }
  
  display.display();
}

// HTTP API endpoint returning JSON data
void handleGetTelemetry() {
  String json = "{\"db\":" + String(currentDB) + 
                 ",\"threshold\":" + String(soundThresholdDB) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update threshold
void handleSetThreshold() {
  if (server.hasArg("threshold")) {
    soundThresholdDB = server.arg("threshold").toInt();
    Serial.printf("[Server] Threshold updated to %d dB.\n", soundThresholdDB);
    updateOLED();
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Sound Level Meter</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .db-val { font-size: 64px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .db-val.alarm { color: #ef4444; animation: pulse 1s infinite; }\n";
  html += "  @keyframes pulse { 0%, 100% { transform: scale(1); } 50% { transform: scale(1.05); } }\n";
  html += "  .slider-container { margin: 30px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 10px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Sound Level Monitor</h1>\n";
  html += "  <div id=\"dbDisplay\" class=\"db-val\">30 dB</div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"threshRange\">Alarm Threshold: <span id=\"threshVal\">75</span> dB</label>\n";
  html += "    <input type=\"range\" id=\"threshRange\" min=\"30\" max=\"100\" value=\"" + String(soundThresholdDB) + "\" oninput=\"updateThreshold(this.value)\">\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 250 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function fetchTelemetry() {\n";
  html += "    fetch('/api/telemetry')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const display = document.getElementById('dbDisplay');\n";
  html += "        display.innerText = data.db + ' dB';\n";
  
  html += "        if (data.db > data.threshold) {\n";
  html += "          display.classList.add('alarm');\n";
  html += "        } else {\n";
  html += "          display.classList.remove('alarm');\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function updateThreshold(val) {\n";
  html += "    document.getElementById('threshVal').innerText = val;\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('threshold', val);\n";
  html += "    fetch('/api/threshold', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    fetchTelemetry();\n";
  html += "    setInterval(fetchTelemetry, 250); // Poll 4 times a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  // Initialize SSD1306 OLED
  if (!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println(R"(SSD1306 allocation failed)");
    for (;;);
  }
  display.clearDisplay();
  display.display();
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/telemetry", handleGetTelemetry);
  server.on("/api/threshold", HTTP_POST, handleSetThreshold);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read sound sensor and map to decibels
  int rawVal = analogRead(SOUND_PIN);
  currentDB = calculateDecibels(rawVal);
  
  // Toggle warning LED based on active threshold comparison
  if (currentDB > soundThresholdDB) {
    digitalWrite(LED_PIN, HIGH);
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  // Refresh local screen display
  static unsigned long lastOledUpdate = 0;
  if (millis() - lastOledUpdate >= 250) {
    updateOLED();
    lastOledUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the sound sensor), **SSD1306 OLED**, and **LED** onto the canvas.
2. Wire Potentiometer to **GPIO34**, OLED to **GPIO21/GPIO22**, and LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer widget. Watch the decibel readout change on the OLED screen and the webpage.
5. Move the threshold slider on the webpage and verify that the Red LED turns ON when the sound level exceeds the set threshold.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Server] Threshold updated to 65 dB.
```

OLED Display:
```
DB LEVEL MONITOR
----------------
78 dB
Threshold: 65 dB
ALARM: OVER LIMIT!
```

## Expected Canvas Behavior
* Adjusting the potentiometer updates the decibel value on both the OLED and webpage.
* If the value exceeds the threshold slider, the Red LED widget turns ON immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `calculateDecibels(...)` | Maps raw ADC voltage readings (0-4095) to approximate decibel metrics. |
| `server.arg("threshold")` | Extracts the threshold value sent via POST request payload. |
| `display.printf(...)` | Formats and prints telemetry logs onto the screen. |
| `setInterval(fetchTelemetry, 250)` | Configures high-frequency client-side polling to track noise spikes. |

## Hardware & Safety Concept: Peak Detection and Signal Amplification
Sound sensors use electret microphone elements that output high-frequency AC signals centered around half the VCC voltage (e.g. 1.65V). A raw ADC read captures the signal at a single point in time, which can miss the actual sound peaks. To obtain accurate sound levels:
1. **Preamplifier circuit**: Use an op-amp envelope detector circuit to filter out the AC carrier wave and output a smooth analog envelope curve.
2. **Software Peak Sampling**: Sample the ADC pin in a rapid loop for 50 ms to capture the absolute maximum voltage peak before mapping.

## Try This! (Challenges)
1. **Sound Level Bar Graph**: Renders a vertical level bar graph on the OLED representing sound intensity.
2. **Pulse alarm buzzer**: Add a buzzer (GPIO 15) and sound a rapid pulse beep if the sound exceeds the threshold.
3. **SPIFFS storage**: Store the threshold configuration value in the SPIFFS filesystem so that it persists across reboots.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The OLED display remains blank | I2C address mismatch | Verify the I2C address in `display.begin()` matches your module (usually `0x3C`) |
| Sound levels fluctuate wildly | Peak sampling missing | Implement a software moving-average filter or sample peak envelopes |
| Threshold changes do not save | POST route error | Verify that the webpage Javascript sends the POST parameters matching the server route |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [36 - ESP32 OLED Digital Clock NTP synced](../beginner/36-esp32-oled-digital-clock-ntp-synced.md)
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [97 - ESP32 Web-controlled Joystick Servo (Pan & Tilt)](97-esp32-web-controlled-joystick-servo-pan-tilt.md) (Next project)
