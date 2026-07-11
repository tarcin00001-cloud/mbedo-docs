# 167 - Current Overload Contactor Breaker IoT Dashboard (Reset online)

Build an internet-connected smart circuit breaker on the ESP32 that monitors load current using an ACS712 sensor on GPIO 34, controls a power contactor relay on GPIO 12, sounds a buzzer alarm on GPIO 15, and hosts a web dashboard displaying live current telemetry, adjustable trip limits, and a remote manual reset button.

## Goal
Learn how to read current sensors, implement digital latching trip logic, implement debounced overload integration to ignore startup current spikes, control high-power relays, and build remote reset panels.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An ACS712 current sensor is on GPIO 34, a power relay on GPIO 12, and a buzzer on GPIO 15. The ESP32 monitors load current. If current exceeds a set limit (e.g. 800 mA) for more than 500 ms, the system trips: it opens the relay instantly to isolate the load, sounds the buzzer, and enters a locked trip state. Navigating to the ESP32's IP address displays status indicators, a trip limit slider, and a Reset button to clear the fault.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor Module | `potentiometer` | Yes | Yes |
| Relay Module (Contactor simulation) | `relay` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the load current. Turning it high simulates an electrical overload event.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT (Analog) | GPIO34 | Yellow | Load current signal |
| Relay Module | IN (Signal) | GPIO12 | Blue | Contactor switch |
| Buzzer | Positive (+) | GPIO15 | Red | Alarm audio indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the ACS712 module from 5V Vin to match its internal sensor amplification calibration, but clamp the analog output to 3.3V using a voltage divider before wiring to the ESP32 pin.

## Code
```cpp
// Current Overload Contactor Breaker IoT (ACS712 sensor + Safety Trip state + Web Reset control)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int CURRENT_PIN = 34;
const int RELAY_PIN = 12;
const int BUZZER_PIN = 15;

WebServer server(80);

// Sensor Calibration values
const int ADC_CENTER_OFFSET = 2048; // ACS712 zero-current Vcc/2 offset
const double CURRENT_SENSITIVITY = 0.185; // 185 mV/A for 5A model

// Overload Protection parameters
float loadCurrentMa = 0.0;
float tripLimitMa = 800.0; // Current limit threshold (mA)
bool isTripped = false;

unsigned long overloadStartTime = 0;
const unsigned long OVERLOAD_DEBOUNCE_MS = 500; // Ignore spikes under 500 ms

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 100; // Sample every 100 ms

void soundBeep(int freq, int duration) {
  tone(BUZZER_PIN, freq);
  delay(duration);
  noTone(BUZZER_PIN);
}

// Read current sensor and evaluate trip logic
void updateBreakerTelemetry() {
  unsigned long now = millis();
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    
    // 1. Read current
    int rawVal = analogRead(CURRENT_PIN);
    double voltageDiff = ((double)(rawVal - ADC_CENTER_OFFSET) / 4095.0) * 3.3;
    double currentAmps = voltageDiff / CURRENT_SENSITIVITY;
    
    loadCurrentMa = abs(currentAmps * 1000.0);
    if (loadCurrentMa < 8.0) loadCurrentMa = 0.0; // Filter trace noise
    
    // 2. Trip Logic Evaluation
    if (!isTripped) {
      if (loadCurrentMa > tripLimitMa) {
        if (overloadStartTime == 0) {
          overloadStartTime = millis(); // Start timing the overload condition
        } else if (millis() - overloadStartTime >= OVERLOAD_DEBOUNCE_MS) {
          // Trip breaker: cut load power instantly
          isTripped = true;
          digitalWrite(RELAY_PIN, LOW); // Open contactor relay
          Serial.printf("[Breaker] TRIP EVENT! Current %.1f mA exceeded limit %.1f mA.\n", 
                        loadCurrentMa, tripLimitMa);
          soundBeep(500, 800);
        }
      } else {
        overloadStartTime = 0; // Reset overload timer
      }
    }
  }
  
  // 3. Keep relay open and beep alarm if tripped
  if (isTripped) {
    digitalWrite(RELAY_PIN, LOW); // Force contactor open
    
    // Pulse buzzer alarm intermittently
    static unsigned long lastBeepTime = 0;
    if (millis() - lastBeepTime > 1500) {
      lastBeepTime = millis();
      soundBeep(600, 150);
    }
  } else {
    digitalWrite(RELAY_PIN, HIGH); // Closed (Power ON)
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"current\":" + String(loadCurrentMa, 1) + 
                 ",\"limit\":" + String(tripLimitMa, 0) + 
                 ",\"tripped\":" + String(isTripped ? "true" : "false") + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to handle resets and limits
void handlePostControl() {
  if (server.hasArg("action")) {
    String action = server.arg("action");
    if (action == "reset" && isTripped) {
      isTripped = false;
      overloadStartTime = 0;
      digitalWrite(RELAY_PIN, HIGH); // Re-close contactor
      Serial.println("[Breaker] Breaker reset via web dashboard.");
      soundBeep(1200, 100);
      soundBeep(1600, 100);
    }
  }
  if (server.hasArg("limit")) {
    tripLimitMa = server.arg("limit").toFloat();
    Serial.printf("[Breaker Config] Trip limit updated to: %.1f mA\n", tripLimitMa);
  }
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Breaker Panel</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 15px; text-transform: uppercase; margin: 20px 0; background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .status-badge.tripped { background-color: #7f1d1d; color: #fca5a5; animation: blinker 1.5s linear infinite; }\n";
  html += "  @keyframes blinker { 50% { opacity: 0.5; } }\n";
  html += "  .current-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #38bdf8; margin: 15px 0; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #10b981; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; margin-bottom: 20px; }\n";
  html += "  .btn:hover { background-color: #059669; }\n";
  html += "  .btn:disabled { background-color: #334155; color: #64748b; cursor: not-allowed; }\n";
  html += "  .tuning-box { border-top: 2px solid #334155; padding-top: 20px; margin-top: 15px; text-align: left; }\n";
  html += "  .slider-row { display: flex; align-items: center; gap: 15px; margin-bottom: 15px; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; width: 150px; font-size: 13px; }\n";
  html += "  input[type=range] { flex-grow: 1; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 18px; height: 18px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .value-lbl { font-family: monospace; font-size: 15px; font-weight: bold; width: 65px; text-align: right; color: #38bdf8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Contactor Panel</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"status-badge\">BREAKER OK</div>\n";
  html += "  <div id=\"currDisplay\" class=\"current-val\">0 mA</div>\n";
  
  html += "  <button class=\"btn\" id=\"resetBtn\" onclick=\"resetBreaker()\" disabled>Reset Breaker</button>\n";
  
  html += "  <div class=\"tuning-box\">\n";
  html += "    <h3>Trip Calibration</h3>\n";
  // Trip limit slider
  html += "    <div class=\"slider-row\"><label>Trip Limit (mA)</label><input type=\"range\" id=\"limitSldr\" min=\"200\" max=\"1500\" step=\"50\" value=\"" + String(tripLimitMa, 0) + "\" oninput=\"tuneLimit()\"><span id=\"limitVal\" class=\"value-lbl\">" + String(tripLimitMa, 0) + " mA</span></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Contactor Pin: 12 | Trip delay: 500 ms</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('currDisplay').innerText = Math.round(data.current) + ' mA';\n";
  
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  html += "        const btnEl = document.getElementById('resetBtn');\n";
  
  html += "        if (data.tripped) {\n";
  html += "          bdgEl.innerText = 'TRIPPED OVERLOAD';\n";
  html += "          bdgEl.className = 'status-badge tripped';\n";
  html += "          btnEl.disabled = false;\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'BREAKER OK';\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "          btnEl.disabled = true;\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function resetBreaker() {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('action', 'reset');\n";
  html += "    fetch('/api/control', { method: 'POST', body: body })\n";
  html += "      .then(() => updateHUD());\n";
  html += "  }\n";
  
  html += "  let lastSentTime = 0;\n";
  html += "  function tuneLimit() {\n";
  html += "    const limit = document.getElementById('limitSldr').value;\n";
  html += "    document.getElementById('limitVal').innerText = limit + ' mA';\n";
  
  html += "    const now = Date.now();\n";
  html += "    if (now - lastSentTime < 100) return;\n";
  html += "    lastSentTime = now;\n";
  
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('limit', limit);\n";
  
  html += "    fetch('/api/control', { method: 'POST', body: body });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 200); // Fast status poll\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(CURRENT_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, HIGH); // Default CLOSED (Power ON)
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 Smart Breaker Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  server.on("/api/control", HTTP_POST, handlePostControl);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
}

void loop() {
  server.handleClient();
  
  // Update breaker readings and trip loops
  updateBreakerTelemetry();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, **Buzzer**, and a **Potentiometer** (to simulate the ACS712 sensor) onto the canvas.
2. Wire Potentiometer wiper to **GPIO34**, Relay input to **GPIO12**, and Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In normal state, keep the potentiometer slider centered (low/moderate load). Verify that the relay is CLOSED (ON) and status is `BREAKER OK`.
6. Slide the simulated current potentiometer high (above 800 mA). Keep it there for 1 second.
7. Verify that the relay clicks open (OFF) instantly, the buzzer sounds, the web status badge blinks red `TRIPPED OVERLOAD`, and the Reset button becomes active.
8. Click "Reset Breaker" on the webpage. Verify that the breaker does not reset if the potentiometer is still high.
9. Slide the potentiometer back down, then click "Reset Breaker". Verify that the relay closes (ON) and status returns to `BREAKER OK`.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Breaker] TRIP EVENT! Current 920.4 mA exceeded limit 800.0 mA.
[Breaker] Breaker reset via web dashboard.
```

## Expected Canvas Behavior
* Actuating the simulated current potentiometer beyond limits opens the contactor relay widget and triggers buzzer alarm chimes.
* Webpage reset overrides re-close the contactor when current levels are safe.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rawVal - ADC_CENTER_OFFSET` | Subtracts the zero-current reference offset. |
| `loadCurrentMa > tripLimitMa` | Compares active load current against the trip limit. |
| `millis() - overloadStartTime >= OVERLOAD_DEBOUNCE_MS` | Debounces the current reading to ignore motor startup current surges. |
| `isTripped = true` | Locks the breaker into the tripped fault state. |

## Hardware & Safety Concept: Inrush Current Debouncing and Arc Suppression
* **Inrush Current Debouncing**: Capacitive and inductive loads (like switching power supplies or motors) draw high surge currents (inrush current) when first connected, often 5 to 10 times their normal operating current. If the breaker tripped instantly, it would trigger every time the load turned on. The **debouncing delay** (e.g. 500 ms) allows these safe startup surges to pass while still protecting against sustained short circuits.
* **Arc Suppression**: High-power relays switching inductive loads generate intense electrical arcing across their contacts when opening, which can weld the contacts closed. Always place a snubber circuit (a series resistor and capacitor) in parallel with the relay contacts to absorb these sparks.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning sign when tripped.
2. **Auto-reclose routine**: Implement an auto-reclose cycle: try to re-close the breaker automatically 3 times (with 10-second intervals) before locking into the permanent TRIP state.
3. **SPIFFS Trip logger**: Save trip event logs (Time, Current, Limit) to SPIFFS flash memory (Project 165).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Breaker trips immediately upon startup | Trip limit too low | Increase the trip limit slider or adjust the `OVERLOAD_DEBOUNCE_MS` to allow inrush current to clear |
| Current reads incorrect values | Calibration offset wrong | Adjust the `ADC_CENTER_OFFSET` in the code until zero load current reads exactly 0 mA |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [123 - ESP32 Current Safety Cutoff system](../expert/123-energy-safety-monitor.md)
- [138 - ESP32 SD Card Energy Logger Web Status](138-esp32-sd-card-energy-logger-web-status.md)
- [159 - ESP32 Automatic Battery Capacity Tester](159-esp32-automatic-battery-capacity-tester-iot-web-logger.md)
- [168 - Bluetooth to WiFi Keypad Entry Lock Gateway](168-esp32-bluetooth-to-wifi-keypad-entry-lock-gateway.md) (Next project)
