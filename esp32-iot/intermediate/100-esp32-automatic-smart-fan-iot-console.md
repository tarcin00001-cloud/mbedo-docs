# 100 - Automatic Smart Fan IoT Console (DHT22 + DC Motor + LCD + Web controller)

Build an automated smart fan climate console on the ESP32 that samples a DHT22 sensor on GPIO 4, displays readings and motor states on a 16x2 I2C LCD, drives a speed-controlled DC motor on GPIO 13 via LEDC hardware PWM, and hosts a web interface to monitor statistics and adjust temperature thresholds.

## Goal
Learn how to sample DHT22 sensors, generate PWM motor signals using the LEDC peripheral, display system diagnostics on I2C LCDs, serve interactive web control pages, and handle HTTP POST variables.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 4, a DC motor (driven via transistor or L298N) on GPIO 13, and a 16x2 LCD on I2C (GPIO 21/22). The LCD shows live climate data and fan speeds. Navigating to the ESP32's IP address opens a dashboard showing climate telemetry. The webpage has a slider to configure the temperature threshold. If the temperature exceeds the threshold, the fan turns ON and scales its speed dynamically.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| DC Motor (via driver) | `motor_dc` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Climate data signal |
| DC Motor | ENA (Speed Input) | GPIO13 | Orange | PWM speed channel |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the DC motor driver and LCD from the 5V Vin rail.

## Code
```cpp
// Automatic Smart Fan IoT Console (DHT22 climate logic + LEDC DC Motor control)
#include <WiFi.h>
#include <WebServer.h>
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 4;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int MOTOR_PIN = 13;

// LEDC PWM configurations
const int pwmChannel = 0;
const int pwmFreq = 5000;
const int pwmResolution = 8; // 8-bit: 0-255

WebServer server(80);

// Global state variables
float currentTemp = 0.0;
float currentHum = 0.0;
int tempThreshold = 28; // Default threshold set to 28 °C
int activeFanSpeedPercent = 0;

// Convert fan speed byte to percentage
int calculateSpeedPercent(int pwmVal) {
  return map(pwmVal, 0, 255, 0, 100);
}

// Update the 16x2 LCD status screen
void updateLCDDisplay() {
  lcd.clear();
  
  // Row 0: Print Temp and Humidity
  lcd.setCursor(0, 0);
  lcd.printf("T:%.1fC H:%.1f%%", currentTemp, currentHum);
  
  // Row 1: Print active fan speed status
  lcd.setCursor(0, 1);
  if (activeFanSpeedPercent > 0) {
    lcd.printf("Fan Speed: %d%%", activeFanSpeedPercent);
  } else {
    lcd.print("Fan Mode: IDLE");
  }
}

// HTTP API endpoint returning JSON data
void handleGetClimateData() {
  String statusMsg = (activeFanSpeedPercent > 0) ? "COOLING" : "IDLE";
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"hum\":" + String(currentHum, 1) +
                 ",\"speed\":" + String(activeFanSpeedPercent) +
                 ",\"threshold\":" + String(tempThreshold) +
                 ",\"status\":\"" + statusMsg + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update threshold
void handleSetThreshold() {
  if (server.hasArg("threshold")) {
    tempThreshold = server.arg("threshold").toInt();
    Serial.printf("[Server] Threshold updated to %d C.\n", tempThreshold);
    updateLCDDisplay();
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Fan Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 28px; font-weight: bold; margin-top: 5px; font-family: monospace; }\n";
  html += "  .fan-badge { display: inline-block; padding: 10px; width: 100%; box-sizing: border-box; border-radius: 8px; font-weight: bold; font-size: 16px; margin: 10px 0; text-transform: uppercase; }\n";
  html += "  .idle { background-color: #1e293b; color: #64748b; border: 1px solid #334155; }\n";
  html += "  .cooling { background-color: #0369a1; color: #e0f2fe; animation: spin-glow 2s infinite linear; }\n";
  html += "  .slider-container { margin: 25px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 10px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Climate Fan</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temp</div><div class=\"metric-val\" id=\"tempDisplay\">--.- &deg;C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humDisplay\">--.- %</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div id=\"fanBadge\" class=\"fan-badge idle\">FAN: IDLE</div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"threshRange\">Threshold: <span id=\"threshVal\">28</span> &deg;C</label>\n";
  html += "    <input type=\"range\" id=\"threshRange\" min=\"20\" max=\"40\" value=\"" + String(tempThreshold) + "\" oninput=\"updateThreshold(this.value)\">\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateClimateData() {\n";
  html += "    fetch('/api/climate')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  html += "        document.getElementById('humDisplay').innerText = data.hum.toFixed(1) + ' %';\n";
  html += "        const bdgEl = document.getElementById('fanBadge');\n";
  
  html += "        if (data.speed > 0) {\n";
  html += "          bdgEl.innerText = 'FAN ACTIVE: ' + data.speed + '%';\n";
  html += "          bdgEl.className = 'fan-badge cooling';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'FAN IDLE';\n";
  html += "          bdgEl.className = 'fan-badge idle';\n";
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
  html += "    updateClimateData();\n";
  html += "    setInterval(updateClimateData, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Smart Fan Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  // Configure LEDC PWM channel for DC Motor control
  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(MOTOR_PIN, pwmChannel);
  ledcWrite(pwmChannel, 0); // Start with fan OFF
  
  Serial.println("\nESP32 Smart Fan Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/climate", handleGetClimateData);
  server.on("/api/threshold", HTTP_POST, handleSetThreshold);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read temperature and humidity
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  
  if (!isnan(temp) && !isnan(hum)) {
    currentTemp = temp;
    currentHum = hum;
    
    // Automatic temperature speed control logic
    if (currentTemp > tempThreshold) {
      // Calculate speed based on temperature deviation
      // Map temperature range (Threshold to Threshold + 5) to PWM duty cycle (120 to 255)
      int duty = map((int)currentTemp, tempThreshold, tempThreshold + 5, 120, 255);
      if (duty > 255) duty = 255;
      
      ledcWrite(pwmChannel, duty);
      activeFanSpeedPercent = calculateSpeedPercent(duty);
    } else {
      ledcWrite(pwmChannel, 0); // Turn fan OFF
      activeFanSpeedPercent = 0;
    }
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
1. Drag **ESP32 DevKitC**, **DHT22**, **DC Motor**, and **16x2 I2C LCD** onto the canvas.
2. Wire DHT22 to **GPIO4**, DC Motor to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the DHT22 sliders.
5. Move the temperature slider above the threshold configured on the webpage. Verify that the DC motor rotates, its speed is updated on the LCD, and the webpage dashboard updates dynamically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Server] Threshold updated to 26 C.
```

LCD Display:
```
T:29.5C H:45.0%
Fan Speed: 78%
```

## Expected Canvas Behavior
* Adjusting the DHT22 temperature slider above the threshold rotates the DC motor widget at the calculated speed.
* Adjusting the threshold slider on the webpage dynamically updates the fan status and speed.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledcSetup(...)` | Sets up the LEDC PWM channel frequency and resolution. |
| `ledcAttachPin(...)` | Binds the DC motor pin to the LEDC channel. |
| `map(...)` | Scales the temperature deviation to the PWM duty cycle (120 to 255) to prevent stalling. |
| `ledcWrite(...)` | Adjusts the DC motor speed by modifying the duty cycle. |

## Hardware & Safety Concept: Fan Motor Startup Torque and Back EMF
* **Startup Torque**: DC motors require higher starting current to overcome friction. If the PWM duty cycle is too low (e.g. under 80), the motor can stall and draw excessive current, heating up the coils. Set a minimum PWM starting threshold (e.g. 120) to ensure the fan starts reliably.
* **Back EMF Protection**: DC motors are inductive loads. When turned OFF, they generate a high-voltage spike (back EMF). Always install a flyback diode (like a 1N4007) in parallel with the motor terminals to protect the transistor driver.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a graph of temperature changes.
2. **Climate alert buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the temperature exceeds 35.0 °C.
3. **SPIFFS integration**: Store the temperature threshold in the SPIFFS filesystem (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| DC Motor hums but does not rotate | PWM duty cycle too low | Increase the minimum mapping threshold value in the code (e.g. from 120 to 150) |
| LCD shows garbage characters | LCD address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [57 - ESP32 Web Page DC motor speed dial](57-esp32-web-page-dc-motor-speed-dial.md)
- [99 - ESP32 Rain Wiper Servo IoT Control](99-esp32-rain-wiper-servo-iot-control-rain-sensor-servo-web-status.md)
- [101 - ESP32 Automatic Barrier Gate IoT Console](101-esp32-automatic-barrier-gate-iot-console.md) (Next project)
