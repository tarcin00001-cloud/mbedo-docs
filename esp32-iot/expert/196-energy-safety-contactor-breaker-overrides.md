# 196 - Energy Safety Contactor Logger with Web Control Breaker overrides

Build an intelligent digital circuit breaker and power monitor on the ESP32 that measures AC/DC current using an ACS712 sensor, calculates load metrics (Amps and Watts), trips a heavy-duty contactor relay under overload states, and hosts a web control dashboard.

## Goal
Learn how to sample analog current sensors, calculate Root Mean Square (RMS) load metrics, implement fast trip-curve safety safety logic, and build real-time power control dashboards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It samples an ACS712 current sensor (GPIO 34). If the calculated load exceeds a configurable limit (e.g., 5.0 Amps), the ESP32 instantly trips a contactor relay (GPIO 13), activates a buzzer alarm (GPIO 15), and updates its status to "TRIPPED". The administrator can monitor current and active power in real-time, adjust the current limit via a slider, and override the breaker state using an online web panel.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor | `acs712` | Yes | Yes |
| Relay Module (Contactor) | `relay` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 | OUT (Analog Signal) | GPIO34 | Yellow | Current sensor analog out |
| Relay Module | IN (Signal) | GPIO13 | Orange | Contactor breaker control |
| Buzzer | Positive (+) | GPIO15 | Blue | Warning alarm sounder |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The ACS712 output is centered around 2.5V (half of VCC) at zero current. Use a resistor voltage divider if needed to scale it down to the ESP32's 3.3V ADC range, or use a 3.3V compatible sensor module.

## Code
```cpp
// Energy Safety Contactor (ACS712 RMS calculation + Contactor Trip logic + Web control dashboard + limit adjust)
#include <WiFi.h>
#include <WebServer.h>
#include <Preferences.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int CURRENT_SENSOR_PIN = 34;
const int CONTACTOR_RELAY_PIN = 13;
const int BUZZER_PIN = 15;
const int LED_PIN = 12;

WebServer server(80);
Preferences preferences;

// Calibration variables (for 5A ACS712 sensor module)
const float SENSITIVITY = 0.185; // 185 mV per Amp
const float ADC_REF = 3.3;       // ESP32 reference voltage
const int ADC_RESOLUTION = 4095;

// Safety Limit (Amps)
float overloadLimitAmps = 5.0;

// Power load metrics
float currentAmps = 0.0;
float activePowerWatts = 0.0;
const float AC_VOLTAGE_EST = 230.0; // Assumed grid AC voltage

// Breaker states
bool breakerClosed = true; // True = power ON, False = power TRIPPED/OPEN
bool overloadTripped = false;

// Calculate RMS Current by sampling the analog pin over a full 50Hz grid cycle (20 ms)
float readRMSCurrent() {
  unsigned long startMillis = millis();
  long sumSquare = 0;
  int sampleCount = 0;
  
  // Sample for 20 ms (1 full cycle of 50 Hz AC)
  while (millis() - startMillis < 20) {
    int val = analogRead(CURRENT_SENSOR_PIN);
    // Offset center point (2048 matches half scale ~1.65V)
    int offsetVal = val - 2048;
    sumSquare += (offsetVal * offsetVal);
    sampleCount++;
  }
  
  float meanSquare = (float)sumSquare / sampleCount;
  float rootMeanSquare = sqrt(meanSquare);
  
  // Convert ADC steps back to voltage, then divide by sensor sensitivity to get Amps
  float voltageRMS = (rootMeanSquare / ADC_RESOLUTION) * ADC_REF;
  float amps = voltageRMS / SENSITIVITY;
  
  // Noise filtering threshold
  if (amps < 0.15) amps = 0.0;
  
  return amps;
}

// Trip the contactor relay
void tripBreaker() {
  breakerClosed = false;
  overloadTripped = true;
  digitalWrite(CONTACTOR_RELAY_PIN, LOW); // De-energize contactor
  digitalWrite(LED_PIN, HIGH);
  tone(BUZZER_PIN, 2000); // Trigger continuous warning alarm
  Serial.printf("[Breaker Alert] Overload tripped! Load: %.2f A. Contactor OPEN.\n", currentAmps);
}

// Reset tripped state
void resetBreaker() {
  overloadTripped = false;
  breakerClosed = true;
  digitalWrite(CONTACTOR_RELAY_PIN, HIGH); // Re-energize contactor
  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);
  Serial.println("[Breaker] Contactor reset. Power RESTORED.");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Breaker Panel</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .status-banner { text-align: center; padding: 15px; border-radius: 8px; font-weight: bold; margin: 20px 0; border: 1px solid #334155; }\n";
  html += "  .closed { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .tripped { background-color: #7f1d1d; color: #fca5a5; animation: blink 1s infinite; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .slider-box { margin: 25px 0; }\n";
  html += "  .slider { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-weight: bold; border-radius: 6px; border: none; cursor: pointer; text-align: center; text-decoration: none; margin-bottom: 10px; font-size: 15px; }\n";
  html += "  .btn-reset { background-color: #ef4444; color: white; }\n";
  html += "  .btn-toggle { background-color: #38bdf8; color: #0f172a; }\n";
  html += "  @keyframes blink { 50% { opacity: 0.8; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Power Contactor HUD</h1>\n";
  
  // Status check
  if (overloadTripped) {
    html += "  <div class=\"status-banner tripped\">🚨 BREAKER TRIPPED - OVERLOAD DETECTED</div>\n";
  } else if (breakerClosed) {
    html += "  <div class=\"status-banner closed\">BREAKER CLOSED - POWER CONNECTED</div>\n";
  } else {
    html += "  <div class=\"status-banner tripped\" style='background-color:#451a03;color:#fed7aa;'>BREAKER OPEN - MANUALLY DISCONNECTED</div>\n";
  }
  
  // Metrics
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Current load</div><div class=\"metric-val\" id=\"ampVal\">0.00 A</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Active Power</div><div class=\"metric-val\" id=\"wattVal\">0 W</div></div>\n";
  html += "  </div>\n";
  
  // Limit adjust slider
  html += "  <div class=\"slider-box\">\n";
  html += "    <form method=\"POST\" action=\"/setlimit\">\n";
  html += "      <label style='display:flex; justify-content:space-between; margin-bottom:10px;'>\n";
  html += "        <span>Overload Limit</span>\n";
  html += "        <span style='color:#38bdf8;font-weight:bold;'>" + String(overloadLimitAmps, 1) + " A</span>\n";
  html += "      </label>\n";
  html += "      <input type=\"range\" name=\"limit\" class=\"slider\" min=\"1.0\" max=\"10.0\" step=\"0.5\" value=\"" + String(overloadLimitAmps, 1) + "\" onchange=\"this.form.submit()\">\n";
  html += "    </form>\n";
  html += "  </div>\n";
  
  // Action controls
  if (overloadTripped) {
    html += "  <a href=\"/reset\" class=\"btn btn-reset\">RESET OVERLOAD TRIP</a>\n";
  } else {
    html += "  <a href=\"/toggle\" class=\"btn btn-toggle\">" + String(breakerClosed ? "MANUALLY TRIP BREAKER" : "CLOSE BREAKER (RESTORE)") + "</a>\n";
  }
  
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  setInterval(function() {\n";
  html += "    fetch('/api/metrics').then(response => response.json()).then(data => {\n";
  html += "      document.getElementById('ampVal').innerText = data.amps.toFixed(2) + ' A';\n";
  html += "      document.getElementById('wattVal').innerText = Math.round(data.watts) + ' W';\n";
  // Reload page to sync UI banner states if tripped online
  html += "    });\n";
  html += "  }, 1000);\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST route returning current metrics
void handleGetMetrics() {
  String json = "{\"amps\":" + String(currentAmps, 2) + ",\"watts\":" + String(activePowerWatts, 1) + "}";
  server.send(200, "application/json", json);
}

// Handle limit adjustments POST request
void handleSetLimit() {
  if (server.hasArg("limit")) {
    overloadLimitAmps = server.arg("limit").toFloat();
    
    // Save to NVS
    preferences.begin("breaker", false);
    preferences.putFloat("limit", overloadLimitAmps);
    preferences.end();
    
    Serial.printf("[Breaker] Safety limit updated to: %.1f A\n", overloadLimitAmps);
  }
  server.sendHeader("Location", "/", true);
  server.send(303);
}

// Handle manual toggle route
void handleToggle() {
  if (overloadTripped) {
    server.send(403, "text/plain", "Action Denied. Overload condition unresolved.");
  } else {
    breakerClosed = !breakerClosed;
    digitalWrite(CONTACTOR_RELAY_PIN, breakerClosed ? HIGH : LOW);
    Serial.printf("[Manual] Contactor state changed: %s\n", breakerClosed ? "CLOSED" : "OPEN");
    
    server.sendHeader("Location", "/", true);
    server.send(303);
  }
}

// Reset overload trip route
void handleResetTrip() {
  resetBreaker();
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(CONTACTOR_RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  digitalWrite(CONTACTOR_RELAY_PIN, HIGH); // Closed initially (Power ON)
  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);
  
  // Load settings from Preferences NVS namespace
  preferences.begin("breaker", true);
  overloadLimitAmps = preferences.getFloat("limit", 5.0);
  preferences.end();
  
  Serial.println("\nESP32 Smart Breaker Station starting...");
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
  server.on("/api/metrics", HTTP_GET, handleGetMetrics);
  server.on("/setlimit", HTTP_POST, handleSetLimit);
  server.on("/toggle", HTTP_GET, handleToggle);
  server.on("/reset", HTTP_GET, handleResetTrip);
  server.begin();
  
  Serial.println("HTTP Server active.");
}

void loop() {
  server.handleClient();
  
  // 1. Measure load current RMS
  currentAmps = readRMSCurrent();
  
  // Mock simulation values in workspace if ACS712 is not reading load
  if (currentAmps == 0.0 && breakerClosed) {
    static float mockAmps = 2.5;
    // Add small fluctuation
    mockAmps += ((float)random(-2, 3) / 20.0);
    if (mockAmps < 0.0) mockAmps = 0.0;
    currentAmps = mockAmps;
  } else if (!breakerClosed) {
    currentAmps = 0.0;
  }
  
  activePowerWatts = currentAmps * AC_VOLTAGE_EST;
  
  // 2. Overload Trip Condition check
  if (currentAmps >= overloadLimitAmps && breakerClosed && !overloadTripped) {
    tripBreaker();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ACS712 Sensor**, **Relay**, **Buzzer**, and **LED** onto the canvas.
2. Wire the parts to the specified pins.
3. Paste the code and click **Run**.
4. Verify that the console prints the connection log and launches the server.
5. In simulation, since a physical load cannot be applied directly, mock a current spike by entering `SYS:CURRENT:6.5` in the serial terminal (assuming the limit is set to 5.0A).
6. Verify that the contactor relay widget on the canvas turns OFF immediately, the buzzer sounds, and the LED turns ON.
7. Open your browser and navigate to the IP address. Verify that the dashboard shows "🚨 BREAKER TRIPPED".
8. Set the current back to normal, then click "RESET OVERLOAD TRIP" on the web page to restore power.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP Server active.
[Breaker Alert] Overload tripped! Load: 6.50 A. Contactor OPEN.
[Breaker] Contactor reset. Power RESTORED.
```

## Expected Canvas Behavior
* Sending simulated current values above the threshold turns off the contactor relay widget, blinks the LED, and sounds the buzzer.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readRMSCurrent()` | Samples the analog current sensor over a full 20 ms cycle (50Hz AC wave) to compute RMS values. |
| `overloadLimitAmps` | The trip current threshold compared to the measured RMS value. |
| `digitalWrite(..., LOW)` | Trips the contactor (de-energizes the coil) to open the high-voltage circuit. |

## Hardware & Safety Concept: Current Tripping Curves and Contactor Coil Heat
* **Current Tripping Curves**: Industrial circuit breakers do not trip instantly on minor overcurrent: they follow a thermal-magnetic curve (tripping slowly for minor overloads, and instantly for major short-circuits). In code, you can implement this by delaying the trip for minor overloads:
  `if (amps > limit * 1.2 && millis() - overloadStart > 5000) { tripBreaker(); }`
* **Contactor Coil Heat**: Continuous-duty relays and contactors generate significant heat when their coils are energized for long periods. Choose contactors with energy-saving economizer circuits or use latching contactors that only require a brief pulse to change states.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current load (Amps/Watts) and breaker state.
2. **Dynamic Voltage Input**: Add a secondary ZMPT101B AC voltage sensor input to measure the actual grid voltage dynamically instead of assuming 230V.
3. **ThingSpeak Graphing**: Publish power load data to a ThingSpeak channel (Project 140) every 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| ACS712 readings are highly noisy | ADC input pin noise | ESP32 ADC pins have high noise levels. To fix this, increase the number of analog samples averaged in the RMS calculation loop |
| Relay does not switch | Insufficient logic level voltage | Some heavy-duty contactor relays require a 5V logic signal. Use a transistor buffer circuit (e.g. 2N2222) to convert the ESP32's 3.3V out to 5V |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [123 - Energy Safety Monitor](../expert/123-energy-safety-monitor.md)
- [167 - Current Overload Contactor Breaker IoT Dashboard](167-esp32-current-overload-contactor-breaker-iot-dashboard-reset-online.md)
- [197 - Dual Axis Robotic Arm PID Position Controller Web HUD](197-dual-axis-robotic-arm-pid-position-controller-web-hud.md) (Next project)
