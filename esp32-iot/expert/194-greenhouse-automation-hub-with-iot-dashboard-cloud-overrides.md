# 194 - Greenhouse Automation Hub with IoT Dashboard & Cloud overrides

Build a smart greenhouse automation hub on the ESP32 that monitors climate parameters (DHT22, soil moisture, LDR), controls actuators (water pump, fan, and growth lights), and hosts an interactive local dashboard with manual/automatic operating modes.

## Goal
Learn how to design multi-zone hysteresis control loops, write asynchronous HTTP override endpoints, save states to NVS preferences, scale multi-sensor ADC maps, and code clean agricultural HUDs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It samples ambient climate (DHT22 on GPIO 14), soil moisture (GPIO 34), and light levels (GPIO 35). Under Auto Mode, the system automatically triggers a water pump relay (GPIO 13) if soil moisture drops below 40%, turns on a cooling fan (GPIO 27) if temperature exceeds 28 °C, and activates growth lights (GPIO 12) if the room goes dark. Navigating to the IP address opens an executive dashboard where the user can view charts, toggle manual override switches, and command relays directly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Soil Moisture Sensor | `soil` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| 3x Relay Modules | `relay` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| Soil Moisture | AO (Analog Out) | GPIO34 | Green | Soil moisture analog input |
| LDR | AO (Analog Out) | GPIO35 | Blue | Light sensor analog input |
| Pump Relay | IN (Signal) | GPIO13 | Orange | Water pump control |
| Fan Relay | IN (Signal) | GPIO27 | Violet | Exhaust fan control |
| Light Relay | IN (Signal) | GPIO12 | Red | Growth lights control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect series resistors where applicable, and run separate external power rails for heavy relay coils to protect the ESP32.

## Code
```cpp
// Greenhouse Automation Hub (DHT22 + Soil + LDR + 3x Relays + NVS Preferences + Web HUD Console)
#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int SOIL_PIN = 34;
const int LDR_PIN = 35;

// Relay Output Pins
const int PUMP_RELAY = 13;
const int FAN_RELAY = 27;
const int LIGHT_RELAY = 12;

WebServer server(80);
Preferences preferences;

// Environmental readings
float tempC = 0.0;
float humidPct = 0.0;
float soilMoisturePct = 0.0;
float lightPct = 0.0;

// System states
bool autoMode = true; // True = Automatic hysteresis, False = Manual web controls
bool pumpState = false;
bool fanState = false;
bool lightState = false;

// Hysteresis safety limits
const float TEMP_LIMIT_HIGH = 28.0;   // Turn fan ON above 28 C
const float TEMP_LIMIT_LOW = 26.0;    // Turn fan OFF below 26 C
const float SOIL_LIMIT_LOW = 40.0;    // Turn pump ON below 40% moisture
const float SOIL_LIMIT_HIGH = 60.0;   // Turn pump OFF above 60% moisture
const float LIGHT_LIMIT_LOW = 30.0;   // Turn lights ON below 30% light

// Fetch climate readings and run automated rule checking
void processGreenhouseControls() {
  // Read sensors
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  if (!isnan(t) && !isnan(h)) {
    tempC = t;
    humidPct = h;
  }
  
  int rawSoil = analogRead(SOIL_PIN);
  // Map raw soil reading (4095 dry -> 0 wet) to percentage
  soilMoisturePct = (1.0 - (rawSoil / 4095.0)) * 100.0;
  
  int rawLight = analogRead(LDR_PIN);
  lightPct = (rawLight / 4095.0) * 100.0;
  
  // Apply automated threshold rules if Auto Mode is enabled
  if (autoMode) {
    // 1. Fan Control (Temperature Hysteresis)
    if (tempC >= TEMP_LIMIT_HIGH && !fanState) {
      fanState = true;
      digitalWrite(FAN_RELAY, HIGH);
      Serial.println("[Auto Mode] High temperature detected. Fan turned ON.");
    } else if (tempC <= TEMP_LIMIT_LOW && fanState) {
      fanState = false;
      digitalWrite(FAN_RELAY, LOW);
      Serial.println("[Auto Mode] Temperature normal. Fan turned OFF.");
    }
    
    // 2. Pump Control (Soil Moisture Hysteresis)
    if (soilMoisturePct <= SOIL_LIMIT_LOW && !pumpState) {
      pumpState = true;
      digitalWrite(PUMP_RELAY, HIGH);
      Serial.println("[Auto Mode] Dry soil detected. Water pump turned ON.");
    } else if (soilMoisturePct >= SOIL_LIMIT_HIGH && pumpState) {
      pumpState = false;
      digitalWrite(PUMP_RELAY, LOW);
      Serial.println("[Auto Mode] Soil moist. Water pump turned OFF.");
    }
    
    // 3. Growth Lights Control
    if (lightPct <= LIGHT_LIMIT_LOW && !lightState) {
      lightState = true;
      digitalWrite(LIGHT_RELAY, HIGH);
      Serial.println("[Auto Mode] Low light detected. Growth lights turned ON.");
    } else if (lightPct > LIGHT_LIMIT_LOW && lightState) {
      lightState = false;
      digitalWrite(LIGHT_RELAY, LOW);
      Serial.println("[Auto Mode] Ambient light sufficient. Lights turned OFF.");
    }
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Greenhouse HUD Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 580px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .mode-switch { text-align: center; margin-bottom: 25px; }\n";
  html += "  .btn-mode { display: inline-block; padding: 8px 20px; font-weight: bold; border-radius: 20px; border: 1px solid #334155; background-color: #0f172a; color: #94a3b8; cursor: pointer; text-decoration: none; }\n";
  html += "  .btn-mode.active { background-color: #38bdf8; color: #0f172a; border-color: #38bdf8; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .control-row { display: flex; justify-content: space-between; align-items: center; padding: 15px 0; border-bottom: 1px solid #1e293b; }\n";
  html += "  .btn-relay { padding: 10px 20px; font-weight: bold; border-radius: 6px; border: none; cursor: pointer; text-decoration: none; display: inline-block; }\n";
  html += "  .btn-on { background-color: #10b981; color: white; }\n";
  html += "  .btn-off { background-color: #ef4444; color: white; }\n";
  html += "  .btn-disabled { background-color: #334155; color: #64748b; cursor: not-allowed; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Greenhouse Control Panel</h1>\n";
  
  // Mode switcher tabs
  html += "  <div class=\"mode-switch\">\n";
  html += "    <a href=\"/setmode?mode=auto\" class=\"btn-mode " + String(autoMode ? "active" : "") + "\">AUTOMATIC</a>\n";
  html += "    <a href=\"/setmode?mode=manual\" class=\"btn-mode " + String(!autoMode ? "active" : "") + "\">MANUAL OVERRIDE</a>\n";
  html += "  </div>\n";
  
  // Metrics Grid
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempVal\">0.0 C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humVal\">0.0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Soil Moisture</div><div class=\"metric-val\" id=\"soilVal\">0.0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Ambient Light</div><div class=\"metric-val\" id=\"lightVal\">0.0%</div></div>\n";
  html += "  </div>\n";
  
  // Actuator Buttons
  html += "  <h3>Actuator Control Panel</h3>\n";
  
  // Water Pump control
  html += "  <div class=\"control-row\">\n";
  html += "    <span>Irrigation Pump</span>\n";
  if (autoMode) {
    html += "    <span class=\"btn-relay btn-disabled\">" + String(pumpState ? "ON (AUTO)" : "OFF (AUTO)") + "</span>\n";
  } else {
    html += "    <a href=\"/toggle?actuator=pump\" class=\"btn-relay " + String(pumpState ? "btn-on" : "btn-off") + "\">" + String(pumpState ? "ON" : "OFF") + "</a>\n";
  }
  html += "  </div>\n";
  
  // Fan control
  html += "  <div class=\"control-row\">\n";
  html += "    <span>Exhaust Fan</span>\n";
  if (autoMode) {
    html += "    <span class=\"btn-relay btn-disabled\">" + String(fanState ? "ON (AUTO)" : "OFF (AUTO)") + "</span>\n";
  } else {
    html += "    <a href=\"/toggle?actuator=fan\" class=\"btn-relay " + String(fanState ? "btn-on" : "btn-off") + "\">" + String(fanState ? "ON" : "OFF") + "</a>\n";
  }
  html += "  </div>\n";
  
  // Light control
  html += "  <div class=\"control-row\">\n";
  html += "    <span>Growth Lights</span>\n";
  if (autoMode) {
    html += "    <span class=\"btn-relay btn-disabled\">" + String(lightState ? "ON (AUTO)" : "OFF (AUTO)") + "</span>\n";
  } else {
    html += "    <a href=\"/toggle?actuator=light\" class=\"btn-relay " + String(lightState ? "btn-on" : "btn-off") + "\">" + String(lightState ? "ON" : "OFF") + "</a>\n";
  }
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Mode settings saved in NVS Flash</p>\n";
  html += "</div>\n";
  
  // AJAX data fetcher
  html += "<script>\n";
  html += "  function updateDashboard() {\n";
  html += "    fetch('/api/data')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempVal').innerText = data.temp.toFixed(1) + ' C';\n";
  html += "        document.getElementById('humVal').innerText = data.hum.toFixed(1) + '%';\n";
  html += "        document.getElementById('soilVal').innerText = data.soil.toFixed(1) + '%';\n";
  html += "        document.getElementById('lightVal').innerText = data.light.toFixed(1) + '%';\n";
  // Reload page to reflect state changes if mode changes
  html += "      });\n";
  html += "  }\n";
  html += "  window.onload = function() {\n";
  html += "    updateDashboard();\n";
  html += "    setInterval(updateDashboard, 1500);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST route returning JSON data
void handleGetData() {
  String json = "{\"temp\":" + String(tempC, 2) + 
                 ",\"hum\":" + String(humidPct, 1) + 
                 ",\"soil\":" + String(soilMoisturePct, 1) + 
                 ",\"light\":" + String(lightPct, 1) + "}";
  server.send(200, "application/json", json);
}

// Route to change between auto/manual modes
void handleSetMode() {
  if (server.hasArg("mode")) {
    String mode = server.arg("mode");
    autoMode = (mode == "auto");
    
    // Save selection permanently
    preferences.begin("greenhouse", false);
    preferences.putBool("automode", autoMode);
    preferences.end();
    
    Serial.printf("[System] Mode changed to: %s\n", autoMode ? "AUTOMATIC" : "MANUAL");
    
    // If swapping to auto, run calculation loop instantly to sync outputs
    if (autoMode) {
      processGreenhouseControls();
    }
  }
  server.sendHeader("Location", "/", true);
  server.send(303);
}

// Route to toggle actuators manually
void handleToggle() {
  if (!autoMode && server.hasArg("actuator")) {
    String actuator = server.arg("actuator");
    
    if (actuator == "pump") {
      pumpState = !pumpState;
      digitalWrite(PUMP_RELAY, pumpState ? HIGH : LOW);
      Serial.printf("[Manual] Pump toggled to: %d\n", pumpState);
    } 
    else if (actuator == "fan") {
      fanState = !fanState;
      digitalWrite(FAN_RELAY, fanState ? HIGH : LOW);
      Serial.printf("[Manual] Fan toggled to: %d\n", fanState);
    } 
    else if (actuator == "light") {
      lightState = !lightState;
      digitalWrite(LIGHT_RELAY, lightState ? HIGH : LOW);
      Serial.printf("[Manual] Light toggled to: %d\n", lightState);
    }
  }
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  pinMode(PUMP_RELAY, OUTPUT);
  pinMode(FAN_RELAY, OUTPUT);
  pinMode(LIGHT_RELAY, OUTPUT);
  
  // Set all relays OFF initially
  digitalWrite(PUMP_RELAY, LOW);
  digitalWrite(FAN_RELAY, LOW);
  digitalWrite(LIGHT_RELAY, LOW);
  
  // Load settings from Preferences NVS namespace
  preferences.begin("greenhouse", true);
  autoMode = preferences.getBool("automode", true);
  preferences.end();
  
  Serial.println("\nESP32 Greenhouse Hub starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Web Server Routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/data", HTTP_GET, handleGetData);
  server.on("/setmode", HTTP_GET, handleSetMode);
  server.on("/toggle", HTTP_GET, handleToggle);
  server.begin();
  
  Serial.println("HTTP Diagnostic Server active.");
}

void loop() {
  server.handleClient();
  
  // Read sensors and evaluate control loops every 2 seconds
  static unsigned long lastCheck = 0;
  if (millis() - lastCheck > 2000) {
    lastCheck = millis();
    processGreenhouseControls();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **Soil Moisture Sensor**, **LDR**, and **3 Relays** onto the canvas.
2. Wire the parts to the specified pins.
3. Paste the code and click **Run**.
4. Verify that the console prints the connection log and launches the server.
5. Open your browser and navigate to the IP address. Verify that the dashboard displays all climate values.
6. Click "MANUAL OVERRIDE". Verify that the relay control buttons become clickable (changing colors from gray).
7. Click the "Pump ON" button and verify that the corresponding pump relay widget changes state on the canvas.
8. Click "AUTOMATIC" mode. Modify the simulated soil moisture slider to 10% (dry). Verify that the pump relay turns ON automatically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP Diagnostic Server active.
[Auto Mode] Dry soil detected. Water pump turned ON.
[System] Mode changed to: MANUAL
[Manual] Fan toggled to: 1
```

## Expected Canvas Behavior
* Booting the system launches the control cycles. Modifying parameters via the sliders or clicking the dashboard toggles the relay widgets on the canvas.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `(1.0 - (rawSoil / 4095.0)) * 100.0` | Inverts and maps the analog raw values to a percentage (0% dry, 100% wet). |
| `TEMP_LIMIT_HIGH` / `TEMP_LIMIT_LOW` | Defines the threshold range (hysteresis) to prevent rapid relay switching. |
| `preferences.putBool(...)` | Persists the manual/automatic mode preference configuration to flash memory. |

## Hardware & Safety Concept: Control Hysteresis and High Current Isolation
* **Control Hysteresis**: In temperature controls, a simple switch threshold (e.g. turn fan ON if Temp > 28 °C, else OFF) causes the fan to switch ON/OFF rapidly (chatter) if the reading fluctuates between 27.9 °C and 28.1 °C. This damages mechanical relays. Implement hysteresis by using separate ON (28.0 °C) and OFF (26.0 °C) thresholds.
* **High Current Isolation**: High-current AC inductive loads (like water pumps or fans) generate massive flyback voltage spikes when switched OFF. Always use optoisolated relay modules and wire flyback diodes or snubber circuits across the AC contacts to protect the microcontroller.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the active mode and relay states.
2. **Dynamic Limit Tuning**: Create web form fields allowing users to adjust the temperature and moisture thresholds online, saving values to NVS.
3. **ThingSpeak Integration**: Publish all greenhouse data to a remote ThingSpeak channel (Project 140) every 30 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relays fail to switch | Insufficient relay coil power | Relay coils require 5V to switch. Ensure you do not power 5V relays directly from the ESP32's 3.3V out pin |
| Soil readings fluctuate wildly | Electrolysis corrosion | Continuous DC voltage across soil moisture probes causes rapid electrolysis. Power the sensor VCC through an ESP32 GPIO pin, turning it ON only during the brief analog read cycle |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [100 - Automatic Smart Fan IoT Console](../intermediate/100-automatic-smart-fan-iot-console.md)
- [110 - Soil Moisture Automated Irrigation IoT](../intermediate/110-soil-moisture-automated-irrigation-iot.md)
- [195 - Medical Cold Chain Monitor with Alarm Latch & Email Warnings](195-medical-cold-chain-monitor-with-alarm-latch-email-warnings.md) (Next project)
