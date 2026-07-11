# 104 - Night Light Controller IoT (LDR + PIR + Relay + Web override)

Build an automated smart lighting controller on the ESP32 that samples an analog photoresistor (LDR) on GPIO 34 and a PIR motion sensor on GPIO 14, triggers a light relay on GPIO 13 when motion is detected in the dark, and hosts a web dashboard with manual override buttons.

## Goal
Learn how to sample multiple sensors, implement complex conditional control logic (AND conditions), write non-blocking timer shutoffs, handle web override overrides, and process POST configuration requests.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LDR is on GPIO 34, a PIR sensor on GPIO 14, and a relay on GPIO 13. By default (Auto mode), the relay closes (turning ON the light) for 10 seconds only when it is dark and motion is detected. Navigating to the ESP32's IP address displays sensor values and allows forcing the light ON or OFF manually.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Photoresistor (LDR) | `photoresistor` | Yes | Yes |
| 10 kΩ Resistor (voltage divider) | `resistor` | No | Yes |
| PIR Motion Sensor | `button` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the LDR, and a pushbutton (active-HIGH) to simulate the PIR motion sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Photoresistor | Pin 1 / Pin 2 | GPIO34 / 3V3 | Yellow / Red | LDR analog input |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO34 / GND | Black | Divider pull-down resistor |
| PIR Sensor | OUT (Signal) | GPIO14 | Green | Motion detection input |
| Relay Module | IN (Signal) | GPIO13 | Orange | Light control switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the LDR to GPIO 34 and the PIR sensor to GPIO 14. Power the relay and PIR sensor from the 5V Vin rail.

## Code
```cpp
// Night Light Controller IoT (LDR & PIR triggering + Web overrides)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LDR_PIN = 34;
const int PIR_PIN = 14;
const int RELAY_PIN = 13;

WebServer server(80);

// Control Modes
enum OperatingMode { MODE_AUTO, MODE_FORCE_ON, MODE_FORCE_OFF };
OperatingMode currentMode = MODE_AUTO;

int ldrPercent = 0;
bool motionActive = false;
bool lightState = false;

unsigned long lastMotionTime = 0;
const unsigned long LIGHT_ON_DURATION_MS = 10000; // Keep light on for 10 seconds

// Get active operating mode string
String getModeString() {
  if (currentMode == MODE_AUTO) return "AUTO";
  if (currentMode == MODE_FORCE_ON) return "FORCE ON";
  return "FORCE OFF";
}

// HTTP API endpoint returning JSON data
void handleGetLightData() {
  String json = "{\"ldr\":" + String(ldrPercent) + 
                 ",\"motion\":" + String(motionActive ? 1 : 0) +
                 ",\"mode\":\"" + getModeString() + "\"" +
                 ",\"relay\":" + String(lightState ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update mode
void handleSetMode() {
  if (server.hasArg("mode")) {
    String modeArg = server.arg("mode");
    if (modeArg == "AUTO") {
      currentMode = MODE_AUTO;
    } else if (modeArg == "ON") {
      currentMode = MODE_FORCE_ON;
    } else if (modeArg == "OFF") {
      currentMode = MODE_FORCE_OFF;
    }
    Serial.printf("[Server] Mode updated to %s.\n", getModeString().c_str());
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Lighting Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 28px; font-weight: bold; margin-top: 5px; font-family: monospace; }\n";
  html += "  .light-badge { display: inline-block; padding: 10px; width: 100%; box-sizing: border-box; border-radius: 8px; font-weight: bold; font-size: 16px; margin: 10px 0; text-transform: uppercase; }\n";
  html += "  .light-off { background-color: #1e293b; color: #64748b; border: 1px solid #334155; }\n";
  html += "  .light-on { background-color: #eab308; color: #78350f; box-shadow: 0 0 15px rgba(234, 179, 8, 0.4); }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 25px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.active { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Lighting Console</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">LDR Intensity</div><div class=\"metric-val\" id=\"ldrDisplay\">0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Motion</div><div class=\"metric-val\" id=\"motDisplay\">Idle</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div id=\"lightBadge\" class=\"light-badge light-off\">LIGHT: OFF</div>\n";
  
  html += "  <div class=\"btn-group\">\n";
  html += "    <button id=\"btnAuto\" class=\"btn\" onclick=\"setMode('AUTO')\">AUTO</button>\n";
  html += "    <button id=\"btnOn\" class=\"btn\" onclick=\"setMode('ON')\">FORCE ON</button>\n";
  html += "    <button id=\"btnOff\" class=\"btn\" onclick=\"setMode('OFF')\">FORCE OFF</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateLightData() {\n";
  html += "    fetch('/api/light')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('ldrDisplay').innerText = data.ldr + '%';\n";
  html += "        document.getElementById('motDisplay').innerText = data.motion === 1 ? 'ACTIVE' : 'Idle';\n";
  html += "        document.getElementById('motDisplay').style.color = data.motion === 1 ? '#ef4444' : '#f1f5f9';\n";
  
  html += "        const bdgEl = document.getElementById('lightBadge');\n";
  html += "        if (data.relay === 1) {\n";
  html += "          bdgEl.innerText = 'LIGHT: ACTIVE';\n";
  html += "          bdgEl.className = 'light-badge light-on';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'LIGHT: OFF';\n";
  html += "          bdgEl.className = 'light-badge light-off';\n";
  html += "        }\n";
  
  // Update button highlights
  html += "        document.getElementById('btnAuto').className = 'btn' + (data.mode === 'AUTO' ? ' active' : '');\n";
  html += "        document.getElementById('btnOn').className = 'btn' + (data.mode === 'FORCE ON' ? ' active' : '');\n";
  html += "        document.getElementById('btnOff').className = 'btn' + (data.mode === 'FORCE OFF' ? ' active' : '');\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function setMode(modeVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('mode', modeVal);\n";
  html += "    fetch('/api/mode', { method: 'POST', body: body })\n";
  html += "      .then(() => updateLightData());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateLightData();\n";
  html += "    setInterval(updateLightData, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LDR_PIN, INPUT);
  pinMode(PIR_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with light OFF
  
  Serial.println("\nESP32 Night Light Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/light", handleGetLightData);
  server.on("/api/mode", HTTP_POST, handleSetMode);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read sensor inputs
  int rawLdr = analogRead(LDR_PIN);
  ldrPercent = map(rawLdr, 0, 4095, 0, 100);
  motionActive = (digitalRead(PIR_PIN) == HIGH);
  
  unsigned long now = millis();
  
  // Enforce selected operating mode
  if (currentMode == MODE_AUTO) {
    // Light triggers only if dark (< 30%) AND motion is active
    bool isDark = (ldrPercent < 30);
    
    if (isDark && motionActive) {
      lightState = true;
      lastMotionTime = now;
      digitalWrite(RELAY_PIN, HIGH);
    } 
    else {
      // Turn light off after hold duration has elapsed
      if (lightState && (now - lastMotionTime >= LIGHT_ON_DURATION_MS)) {
        lightState = false;
        digitalWrite(RELAY_PIN, LOW);
      }
    }
  } 
  else if (currentMode == MODE_FORCE_ON) {
    lightState = true;
    digitalWrite(RELAY_PIN, HIGH);
  } 
  else if (currentMode == MODE_FORCE_OFF) {
    lightState = false;
    digitalWrite(RELAY_PIN, LOW);
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the LDR), **Pushbutton** (configured as active-HIGH to simulate the PIR sensor), and **Relay** onto the canvas.
2. Wire Potentiometer to **GPIO34**, Button to **GPIO14**, and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In Auto mode:
   - Slide the LDR potentiometer below 30% (dark) and click the PIR button. Verify that the Relay turns ON (light active) and shuts off after 10 seconds.
6. Click the `FORCE ON` button on the webpage. Verify that the relay turns ON immediately and stays ON, ignoring the sensors.
7. Click `FORCE OFF` to turn the light OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Server] Mode updated to FORCE ON.
[Server] Mode updated to AUTO.
```

Browser JSON Console (`/api/light`):
```json
{"ldr":15,"motion":1,"mode":"AUTO","relay":1}
```

## Expected Canvas Behavior
* In AUTO mode, triggering both the LDR (low voltage) and PIR (high voltage) inputs actuates the relay widget.
* Clicking `FORCE ON`/`FORCE OFF` on the webpage overrides the sensor states immediately.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `isDark && motionActive` | Multi-sensor logical AND condition required to trigger the light. |
| `now - lastMotionTime >= LIGHT_ON_DURATION_MS` | Non-blocking timer to shut down the light after 10 seconds of inactivity. |
| `currentMode == MODE_FORCE_ON` | Override condition to bypass sensor readings. |
| `server.arg("mode")` | Processes the operating mode parameters sent via POST requests. |

## Hardware & Safety Concept: Manual Overrides and Safety Timeouts
In smart home systems, automated controls must support manual overrides to allow users to override sensor logic (e.g. keeping a room light ON while reading). To ensure safety:
1. **Manual Override Modes**: Keep manual override settings in RAM or EEPROM.
2. **Safety Fallback**: For high-power loads (such as heaters), implement a max runtime safety check that turns off the device after 2 hours even if forced ON, preventing overheating.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a lightbulb icon representing the active status.
2. **Fade Night Light**: Replace the relay with an LED (GPIO 13) and fade it in slowly when motion is detected.
3. **SPIFFS integration**: Store the current mode configuration in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light does not turn ON in Auto mode | Ambient light too high | Ensure LDR reading is below 30% (adjust potentiometer slider) |
| PIR triggers continuously | Warm-up noise | Verify that the PIR sensor output has stabilized |
| Webpage buttons are unresponsive | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - ESP32 LDR light meter remote report](../beginner/48-esp32-ldr-light-meter-remote-report.md)
- [98 - ESP32 Gas Leakage Alarm IoT Node](98-esp32-gas-leakage-alarm-iot-node-mq-2-buzzer-relay-web-alert.md)
- [105 - ESP32 Thermostat Controller IoT](105-esp32-thermostat-controller-iot-ntc-relay-lcd-web-threshold-adjust.md) (Next project)
