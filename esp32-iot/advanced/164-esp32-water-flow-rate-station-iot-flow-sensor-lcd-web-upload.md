# 164 - Water Flow Rate Station IoT (Flow sensor + LCD + Web upload)

Build an environmental water consumption telemetry station on the ESP32 that counts pulse signals from a liquid flow sensor (YF-S201) on GPIO 14 using interrupts, displays the flow rate (L/min) and total volume (Liters) on an I2C LCD 1602 (GPIO 21/22), and uploads metrics to a cloud API over WiFi.

## Goal
Learn how to use external interrupts for high-precision pulse counting, calculate fluid flow rates, track cumulative volumes, drive I2C LCD displays, and post data to cloud REST APIs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A YF-S201 flow sensor is on GPIO 14, an LCD 1602 on I2C, and a buzzer on GPIO 15. The ESP32 measures the frequency of the pulse output from the flow sensor using interrupts. It calculates the flow rate (L/min) and total volume (Liters), and displays the values on the LCD. Every 15 seconds, it posts these values to a cloud database simulator (`http://httpbin.org/post`). A local webpage displays live values and sync logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| YF-S201 Hall Effect Water Flow Sensor | `potentiometer` | Yes | Yes |
| I2C LCD 1602 Display | `lcd_i2c` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the flow rate. Turning it up increases the simulated pulse rate.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Flow Sensor | OUT (Signal) | GPIO14 | Yellow | Pulse output |
| LCD 1602 Display| SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Buzzer | Positive (+) | GPIO15 | Red | Alarm output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The YF-S201 flow sensor requires 5V power, but its output signal line should be clamped to 3.3V using a level shifter or voltage divider to protect the ESP32 pin.

## Code
```cpp
// Water Flow Rate Station IoT (YF-S201 pulse sensor + I2C LCD + Web REST upload)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int FLOW_PIN = 14;
const int BUZZER_PIN = 15;

LiquidCrystal_I2C lcd(0x27, 16, 2);
WebServer server(80);

// Interrupt pulse counters
volatile int pulseCount = 0;

// Flow calibration factors (YF-S201 model: F = 7.5 * Q, where Q is L/min)
const float CALIBRATION_FACTOR = 7.5;

float flowRateLmin = 0.0;
float totalVolumeLiters = 0.0;

unsigned long lastSampleTime = 0;
unsigned long lastUploadTime = 0;
const unsigned long UPLOAD_INTERVAL_MS = 15000; // Upload every 15 seconds

String cloudSyncStatus = "Idle";

// Interrupt Service Routine (ISR) to count flow pulses
void IRAM_ATTR flowPulseISR() {
  pulseCount++;
}

// Compute flow rate and accumulate total volume
void computeWaterFlow() {
  unsigned long now = millis();
  double dt = (double)(now - lastSampleTime) / 1000.0;
  
  if (dt >= 1.0) { // Run calculations every 1 second
    lastSampleTime = now;
    
    // Copy volatile variable safely by disabling interrupts briefly
    noInterrupts();
    int pulses = pulseCount;
    pulseCount = 0;
    interrupts();
    
    // 1. Calculate Flow Rate (L/min): flowRate = (Hz) / Calibration
    flowRateLmin = ((float)pulses / dt) / CALIBRATION_FACTOR;
    
    // Simulation fallback if pot is adjusted in MbedO workspace
    if (flowRateLmin == 0.0) {
      int rawMock = analogRead(FLOW_PIN);
      if (rawMock > 100) {
        flowRateLmin = ((float)rawMock / 4095.0) * 15.0; // Max 15 L/min
      }
    }
    
    // 2. Accumulate Volume (Liters): Liters += (L/min * timeS) / 60
    totalVolumeLiters += (flowRateLmin * dt) / 60.0;
    
    // 3. Update I2C LCD Display
    lcd.setCursor(0, 0);
    lcd.print("Flow: ");
    lcd.print(flowRateLmin, 1);
    lcd.print(" L/min ");
    
    lcd.setCursor(0, 1);
    lcd.print("Total: ");
    lcd.print(totalVolumeLiters, 2);
    lcd.print(" L  ");
  }
}

// Upload telemetry to cloud DB API via HTTP POST
void uploadTelemetryToCloud() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"flowRate\":" + String(flowRateLmin, 1) + 
                         ",\"totalVolume\":" + String(totalVolumeLiters, 2) + 
                         ",\"uptime\":" + String(millis() / 1000) + "}";
                         
    Serial.printf("[Cloud] Pushing payload -> %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      cloudSyncStatus = "Success (Code: " + String(httpResponseCode) + ")";
      Serial.println("[Cloud] Upload successful.");
    } else {
      cloudSyncStatus = "Failed (Code: " + String(httpResponseCode) + ")";
      Serial.printf("[Cloud Error] POST failed: %d\n", httpResponseCode);
    }
    http.end();
  } else {
    cloudSyncStatus = "WiFi Offline";
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"flow\":" + String(flowRateLmin, 1) + 
                 ",\"total\":" + String(totalVolumeLiters, 2) + 
                 ",\"sync\":\"" + cloudSyncStatus + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Flow Rate Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; margin-bottom: 15px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .badge.synced { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Water Flow Monitor</h1>\n";
  html += "  <div id=\"syncBadge\" class=\"badge\">Idle</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Flow Rate</div><div class=\"metric-val\" id=\"flowDisplay\">0.0 L/min</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Total Volume</div><div class=\"metric-val\" id=\"totalDisplay\">0.00 L</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Calibration Factor: 7.5 Hz/(L/min) | Target: httpbin.org</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateFlowHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('flowDisplay').innerText = data.flow.toFixed(1) + ' L/min';\n";
  html += "        document.getElementById('totalDisplay').innerText = data.total.toFixed(2) + ' L';\n";
  
  html += "        const bdgEl = document.getElementById('syncBadge');\n";
  html += "        bdgEl.innerText = data.sync;\n";
  html += "        if (data.sync.startsWith('Success')) {\n";
  html += "          bdgEl.className = 'badge synced';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateFlowHUD();\n";
  html += "    setInterval(updateFlowHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(FLOW_PIN, INPUT_PULLUP);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  // Attach interrupt for counting flow sensor pulses
  attachInterrupt(digitalPinToInterrupt(FLOW_PIN), flowPulseISR, RISING);
  
  // Initialize I2C LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Flow Station");
  lcd.setCursor(0, 1);
  lcd.print("Init sensor...");
  
  Serial.println("\nESP32 Water Flow Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
}

void loop() {
  server.handleClient();
  
  // Run flow computations
  computeWaterFlow();
  
  // Periodically upload data to cloud (every 15 seconds)
  unsigned long now = millis();
  if (now - lastUploadTime >= UPLOAD_INTERVAL_MS) {
    lastUploadTime = now;
    uploadTelemetryToCloud();
  }
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **I2C LCD 1602**, and a **Potentiometer** (to simulate the flow sensor pulse train) onto the canvas.
2. Wire Potentiometer wiper to **GPIO14** and LCD to **GPIO21/22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Set the simulated flow potentiometer slider to 40% (light flow). Verify that the LCD display and webpage update to show flow rate and volume accumulation.
6. Increase the slider to 90% (heavy flow). Verify that the flow rate display updates immediately.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Cloud] Pushing payload -> {"flowRate":8.5,"totalVolume":0.42,"uptime":15}
[Cloud] Upload successful.
```

## Expected Canvas Behavior
* Adjusting the simulated sensor potentiometer updates the character text on the I2C LCD and the webpage telemetry numbers.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `attachInterrupt(...)` | Binds the pulse counting ISR to the flow pin on rising edge triggers. |
| `noInterrupts()` | Disables interrupts briefly to read and reset the pulse counter safely. |
| `flowRateLmin = ...` | Converts the pulse frequency (Hz) to liters per minute. |
| `totalVolumeLiters += ...` | Integrates flow rate over time to track total liquid volume. |

## Hardware & Safety Concept: Interrupt Integrity and Level Shifting
* **Interrupt Integrity**: The pulse counting ISR must be declared with `IRAM_ATTR` to run out of internal RAM, ensuring fast execution and preventing watchdog timeouts under high pulse frequencies.
* **Level Shifting**: Flow sensors contain a Hall Effect sensor that typically operates at 5V, outputting 5V logic pulses. Connecting a 5V signal directly to an ESP32 GPIO pin (which is only 3.3V tolerant) can damage the pin over time. Connect a resistor voltage divider (e.g. 10 kΩ and 15 kΩ) between the flow output and the ESP32 pin to divide the signal to a safe 3V level.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a real-time graph of the flow rate.
2. **Leak detection alarm**: Add a buzzer (GPIO 15) to sound an alarm if flow is detected continuously (>0.5 L/min) for more than 5 minutes (indicating a potential pipe leak).
3. **SPIFFS Daily logs**: Save total daily water consumption to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flow rate displays zero | Incorrect interrupt pin | Verify that the flow signal line is wired to GPIO 14 and matches the configuration in code |
| Total volume drifts | Calibration offset | Calibrate the `CALIBRATION_FACTOR` in code to match your specific sensor model's documentation |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - ESP32 SD Card Energy Logger Web Status](138-esp32-sd-card-energy-logger-web-status.md)
- [159 - ESP32 Automatic Battery Capacity Tester](159-esp32-automatic-battery-capacity-tester-iot-web-logger.md)
- [163 - Water Quality Station IoT](163-esp32-water-quality-station-iot-ph-sensor-temp-lcd-web-upload.md)
- [165 - Smart Door Lock with Web Access Override logs](165-esp32-smart-door-lock-with-web-access-override-logs.md) (Next project)
