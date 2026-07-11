# 105 - Thermostat Controller IoT (NTC + Relay + LCD + Web threshold adjust)

Build an automated thermostat climate controller on the ESP32 that samples an analog NTC thermistor sensor on GPIO 34, displays values on a 16x2 I2C LCD, activates a cooling relay on GPIO 13 when temperature limits are exceeded, and hosts a web page to adjust target threshold boundaries dynamically.

## Goal
Learn how to convert analog thermistor voltages to temperature values, print diagnostics to I2C LCDs, implement closed-loop temperature control logic, serve web control pages, and process HTTP POST configuration variables.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An NTC thermistor is on GPIO 34, a cooling relay on GPIO 13, and a 16x2 I2C LCD on I2C (GPIO 21/22). The LCD shows live temperature values and target limits. Navigating to the ESP32's IP address displays temperature statistics. The webpage has a slider to configure the temperature threshold. If the temperature exceeds the threshold, the relay turns ON (cooling active).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NTC Thermistor | `potentiometer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the thermistor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Thermistor | OUT (Analog) | GPIO34 | Yellow | Temp signal input |
| Relay Module | IN (Signal) | GPIO13 | Orange | Cooling control relay |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the relay and LCD from the 5V Vin rail.

## Code
```cpp
// Thermostat Controller IoT (NTC Thermistor reading + Closed-Loop Relay Control + Web thresholds)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int TEMP_PIN = 34;
const int RELAY_PIN = 13;

WebServer server(80);

double currentTemp = 25.0;
int tempThreshold = 30; // Default threshold set to 30 °C
bool coolingActive = false;

// Calculate temperature in Celsius from raw ADC input
// Matches standard NTC thermistor characteristics
double calculateTemperatureC(int rawAdc) {
  if (rawAdc == 0) return -10.0;
  
  // Simplified conversion for simulation: map raw ADC (0-4095) to (-10 to 80 °C)
  double tempC = -10.0 + ((double)rawAdc * 90.0 / 4095.0);
  return tempC;
}

// Update the 16x2 LCD status screen
void updateLCDDisplay() {
  lcd.clear();
  
  // Row 0: Print Current Temperature
  lcd.setCursor(0, 0);
  lcd.printf("Temp:  %.1f C", currentTemp);
  
  // Row 1: Print Threshold and cooling status
  lcd.setCursor(0, 1);
  if (coolingActive) {
    lcd.printf("Limit:%dC [COOL]", tempThreshold);
  } else {
    lcd.printf("Limit:%dC [IDLE]", tempThreshold);
  }
}

// HTTP API endpoint returning JSON data
void handleGetThermostatData() {
  String statusMsg = coolingActive ? "COOLING" : "IDLE";
  String json = "{\"temp\":" + String(currentTemp, 1) + 
                 ",\"threshold\":" + String(tempThreshold) +
                 ",\"relay\":" + String(coolingActive ? 1 : 0) +
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
  html += "<title>Thermostat Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .temp-display { font-size: 56px; font-weight: 800; font-family: monospace; color: #38bdf8; margin: 20px 0; }\n";
  html += "  .temp-display.alarm { color: #38bdf8; animation: pulse 1.5s infinite; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .badge.idle { background-color: #1e293b; color: #64748b; border: 1px solid #334155; }\n";
  html += "  .badge.cooling { background-color: #0369a1; color: #e0f2fe; }\n";
  html += "  .slider-container { margin: 25px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 10px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 20px; height: 20px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Thermostat Console</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"badge idle\">SYSTEM IDLE</div>\n";
  html += "  <div id=\"tempDisplay\" class=\"temp-display\">--.- &deg;C</div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"threshRange\">Threshold Limit: <span id=\"threshVal\">30</span> &deg;C</label>\n";
  html += "    <input type=\"range\" id=\"threshRange\" min=\"15\" max=\"45\" value=\"" + String(tempThreshold) + "\" oninput=\"updateThreshold(this.value)\">\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateThermostat() {\n";
  html += "    fetch('/api/thermostat')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempDisplay').innerHTML = data.temp.toFixed(1) + ' &deg;C';\n";
  
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        bdgEl.innerText = data.status;\n";
  
  html += "        if (data.relay === 1) {\n";
  html += "          bdgEl.className = 'badge cooling';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge idle';\n";
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
  html += "    updateThermostat();\n";
  html += "    setInterval(updateThermostat, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(TEMP_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Thermostat Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  Serial.println("\nESP32 Thermostat Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/thermostat", handleGetThermostatData);
  server.on("/api/threshold", HTTP_POST, handleSetThreshold);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  
  // Read analog thermistor voltage and calculate temperature
  int rawVal = analogRead(TEMP_PIN);
  currentTemp = calculateTemperatureC(rawVal);
  
  // Closed-loop cooling logic
  if (currentTemp > tempThreshold) {
    coolingActive = true;
    digitalWrite(RELAY_PIN, HIGH); // Turn cooling pump/fan ON
  } else {
    coolingActive = false;
    digitalWrite(RELAY_PIN, LOW);  // Turn cooling pump/fan OFF
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
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the NTC thermistor), **Relay**, and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Relay to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer widget.
5. Move the temperature slider above the threshold configured on the webpage. Verify that the Relay turns ON (cooling active), the status updates on the LCD, and the web dashboard updates dynamically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Server] Threshold updated to 28 C.
```

LCD Display:
```
Temp:  31.5 C
Limit:28C [COOL]
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates temperature changes.
* The relay widget turns ON when the simulated temperature exceeds the threshold slider value set on the webpage.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `calculateTemperatureC(...)` | Maps raw ADC voltage readings (0-4095) to temperature metrics. |
| `currentTemp > tempThreshold` | Closed-loop logic check to turn ON/OFF the cooling relay. |
| `updateLCDDisplay()` | Refreshes the local screen display with active metrics. |
| `server.arg("threshold")` | Processes the threshold configuration parameter sent via POST. |

## Hardware & Safety Concept: Sensor Calibration and Thermostat Fail-Safes
* **Thermistor Calibration**: Thermistors are non-linear resistors. Calculating accurate temperatures requires using the **Steinhart-Hart equation** with calibration coefficients (A, B, C) or a lookup table.
* **Overheat Safety Loop**: In software control loops, a failure (such as a crashed web server or damaged sensor wire) can prevent the cooling system from turning ON, causing fires. Implement a hardware fail-safe that cuts power if the temperature exceeds a maximum threshold (e.g. 50 °C).

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a temperature graph.
2. **Cooling alert buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the temperature exceeds 40.0 °C.
3. **SPIFFS integration**: Store the temperature threshold in the SPIFFS filesystem (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temp readout is inverted | Divider configuration swapped | Verify that the thermistor is connected to the 3.3V rail and the divider resistor is connected to ground |
| Relay does not switch | Relay signal pin miswired | Verify that the relay signal is connected to GPIO 13 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [100 - ESP32 Automatic Smart Fan IoT Console](100-esp32-automatic-smart-fan-iot-console.md)
- [106 - ESP32 Shake Alarm System IoT](106-esp32-shake-alarm-system-iot-vibration-sensor-latch-buzzer-web-alert.md) (Next project)
