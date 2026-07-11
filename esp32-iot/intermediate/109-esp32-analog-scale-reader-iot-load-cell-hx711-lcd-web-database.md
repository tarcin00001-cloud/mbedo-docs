# 109 - Analog Scale Reader IoT (Load Cell + HX711 + LCD + Web Database)

Build a web-connected digital scale on the ESP32 that interfaces with a load cell via an HX711 24-bit ADC amplifier (DT on GPIO 12, SCK on GPIO 14), displays weights on a 16x2 I2C LCD, and hosts a web dashboard with a historical log table and manual Tare control.

## Goal
Learn how to interface HX711 load cell amplifiers, perform scale calibration and taring, display weights on I2C LCDs, serve web dashboards, and parse HTTP POST tare requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A load cell is connected to an HX711 amplifier (DT on GPIO 12, SCK on GPIO 14), and a 16x2 LCD on I2C (GPIO 21/22). The LCD shows live weight values. Navigating to the ESP32's IP address displays the current weight. Clicking "Tare Scale" on the web page resets the zero reference. The web page maintains a table of recent weigh-in logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HX711 Amplifier & Load Cell | `potentiometer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the load cell / HX711 interface.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HX711 Module | DOUT (Data) | GPIO12 | Yellow | Serial data input |
| HX711 Module | PD_SCK (Clock) | GPIO14 | Green | Serial clock output |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the HX711 and LCD from the 5V Vin rail.

## Code
```cpp
// Analog Scale Reader IoT (HX711 Weight Scale + LCD Display + Web Tare)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <HX711.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int HX711_DOUT_PIN = 12;
const int HX711_SCK_PIN = 14;

HX711 scale;
WebServer server(80);

float currentWeight = 0.0;
float calibrationFactor = 420.0; // Calibration factor based on load cell parameters

// Rolling history database (Stores last 5 measurements)
String weightDatabase[5] = {"", "", "", "", ""};
int databaseCount = 0;

void addWeightToDb(float weight) {
  for (int i = 4; i > 0; i--) {
    weightDatabase[i] = weightDatabase[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  weightDatabase[0] = "[" + String(timestampS) + "s] Weight: " + String(weight, 1) + " g";
  
  if (databaseCount < 5) databaseCount++;
  Serial.println("[SCALE] Recorded -> " + weightDatabase[0]);
}

void updateLCDDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Weight Scale:");
  
  lcd.setCursor(0, 1);
  lcd.print(currentWeight, 1);
  lcd.print(" g");
}

// HTTP API endpoint returning JSON data
void handleGetScaleData() {
  String json = "{\"weight\":" + String(currentWeight, 1) + ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + weightDatabase[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to tare scale
void handlePostTare() {
  Serial.println("[Scale] Taring sensor...");
  scale.tare(); // Reset zero reference offset
  server.send(200, "text/plain", "OK");
}

// HTTP POST endpoint to save current weight
void handlePostRecord() {
  addWeightToDb(currentWeight);
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Scale Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .weight-val { font-size: 64px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 15px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn-primary { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .btn-primary:hover { background-color: #0ea5e9; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Weight Scale</h1>\n";
  html += "  <div id=\"weightDisplay\" class=\"weight-val\">0.0 g</div>\n";
  
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn\" onclick=\"tareScale()\">Tare Scale</button>\n";
  html += "    <button class=\"btn btn-primary\" onclick=\"recordWeight()\">Record Weight</button>\n";
  html += "  </div>\n";
  
  html += "  <h3>Weigh-in History</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Logs Database</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateScaleData() {\n";
  html += "    fetch('/api/scale')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('weightDisplay').innerText = data.weight.toFixed(1) + ' g';\n";
  
  html += "        const tableBody = document.getElementById('logTable');\n";
  html += "        tableBody.innerHTML = '';\n";
  html += "        data.logs.forEach(log => {\n";
  html += "          if (log.length > 0) {\n";
  html += "            tableBody.innerHTML += '<tr><td>' + log + '</td></tr>';\n";
  html += "          }\n";
  html += "        });\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function tareScale() {\n";
  html += "    fetch('/api/tare', { method: 'POST' })\n";
  html += "      .then(() => updateScaleData());\n";
  html += "  }\n";
  
  html += "  function recordWeight() {\n";
  html += "    fetch('/api/record', { method: 'POST' })\n";
  html += "      .then(() => updateScaleData());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateScaleData();\n";
  html += "    setInterval(updateScaleData, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Scale Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  // Configure scale pins
  scale.begin(HX711_DOUT_PIN, HX711_SCK_PIN);
  scale.set_scale(calibrationFactor); // Apply calibration parameter
  scale.tare();                       // Reset zero reference
  
  Serial.println("\nESP32 Smart Scale Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/scale", handleGetScaleData);
  server.on("/api/tare", HTTP_POST, handlePostTare);
  server.on("/api/record", HTTP_POST, handlePostRecord);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  
  // Read weight from HX711 (take average of 5 readings to smooth output)
  if (scale.is_ready()) {
    float reading = scale.get_units(5);
    if (reading < 0.0) reading = 0.0; // Filter out negative noise
    currentWeight = reading;
  }
  
  // Refresh LCD screen periodically
  static unsigned long lastLcdUpdate = 0;
  if (millis() - lastLcdUpdate >= 1000) {
    updateLCDDisplay();
    lastLcdUpdate = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the load cell / HX711 output), and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer to **GPIO12** (simulating DOUT) and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer widget.
5. Watch the weight value update on the webpage and the LCD. Click the "Record Weight" button to log weight measurements in the history table.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Scale] Taring sensor...
[SCALE] Recorded -> [12s] Weight: 120.4 g
```

LCD Display:
```
Weight Scale:
120.4 g
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates weight changes.
* Clicking "Record Weight" or "Tare Scale" on the webpage triggers actions immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `scale.begin(...)` | Binds the data and clock lines of the HX711 amplifier. |
| `scale.set_scale(calibrationFactor)` | Applies the calibration factor to map raw 24-bit integers to grams. |
| `scale.tare()` | Subtracts active offsets to define the zero reference. |
| `scale.get_units(5)` | Reads and averages 5 raw inputs from the HX711 ADC. |

## Hardware & Safety Concept: Load Cell Calibration and Zero Offsets (Tare)
* **Calibration factor**: Load cells output tiny analog voltage changes. The HX711 converts these to raw 24-bit integers. To get weight in grams, we divide the raw value by a calibration factor. Calculate this factor by placing a known test weight on the scale and adjusting the factor until the readout matches.
* **Taring (Zero Offset)**: Temperature changes and mounting hardware apply mechanical stress to load cells, causing zero-point drift. Enforce a `scale.tare()` on startup and provide a manual Tare button to reset the zero reference.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a custom scale graphic.
2. **Limit Alert Buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the weight exceeds 500 grams.
3. **SPIFFS database**: Save the logs list to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Weight values fluctuate wildly | Bad wire connection | Ensure the load cell wires (Red, Black, White, Green) are soldered securely to the HX711 module |
| Weight displays as negative | Scale inverted | Call `scale.tare()` or check if the load cell is mounted upside down |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [108 - ESP32 Intrusion Detector Alarm IoT](108-esp32-intrusion-detector-alarm-iot-laser-photoresistor-relay-web-notification.md)
- [110 - ESP32 Soil Moisture Automated Irrigation IoT](110-esp32-soil-moisture-automated-irrigation-iot-soil-sensor-relay-web-logger.md) (Next project)
