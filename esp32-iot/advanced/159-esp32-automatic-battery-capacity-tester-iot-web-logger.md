# 159 - Automatic Battery Capacity Tester IoT Web Logger

Build an automated lithium-ion battery capacity tester on the ESP32 that measures battery cell voltage via analog divider GPIO 34, discharges the cell through a load resistor connected via a relay on GPIO 12, logs discharge curves to an SD card, and hosts a web console to control tests and monitor capacity (mAh).

## Goal
Learn how to read battery voltage dividers, calculate discharge current and accumulated capacity (milliamp-hours), log data to SD cards, serve web controllers, and implement cutoff protection.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A battery is connected to analog GPIO 34 through a voltage divider. A discharge load relay is on GPIO 12. Navigating to the ESP32's IP address displays a control panel with Start and Stop buttons. Clicking Start closes the relay to begin discharging. Every second, the ESP32 calculates capacity: `mAh += (Current * time)`. If the cell voltage drops below 3.0V, the relay opens automatically to prevent damage, and the results are saved to `/discharge.csv`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Battery (Li-Ion 18650 cell) | `potentiometer` | Yes | Yes |
| 5.1 Ω Power Resistor (Discharge Load) | `resistor` | No | Yes |
| Relay Module | `relay` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the discharging battery cell voltage. Turning the slider down simulates the battery draining.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Battery Divider | Center Wiper | GPIO34 | Yellow | Cell voltage signal input |
| Relay Module | IN (Signal) | GPIO12 | Blue | Load switch control |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK / MISO / MOSI | GPIO18 / 19 / 23 | Green / Blue / White | SPI interface |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect two 10 kΩ resistors in series across the battery terminals, with the midpoint wired to GPIO 34 to form a 2:1 voltage divider.

## Code
```cpp
// Automatic Battery Capacity Tester IoT Web Logger (Battery Voltage Monitor + Load relay + Capacity Calc)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <SD.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BATTERY_PIN = 34;
const int LOAD_RELAY_PIN = 12;
const int SD_CS_PIN = 5;
const char* logFilepath = "/discharge.csv";

// Test Parameters
const float CUTOFF_VOLTAGE = 3.0; // Stop discharge at 3.0V for Li-ion cells
const float LOAD_RESISTOR_OHM = 5.1; // 5.1 Ohm discharge resistor

WebServer server(80);

// Global state variables
float batteryVoltage = 4.2;
float dischargeCurrentMa = 0.0;
float accumulatedCapacityMah = 0.0;
bool testActive = false;
bool testCompleted = false;

unsigned long testStartTime = 0;
unsigned long lastSampleTime = 0;
unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log data every 5 seconds

String sdStatus = "Disconnected";

// Read battery voltage through 2:1 divider
float readBatteryVoltage() {
  int rawVal = analogRead(BATTERY_PIN);
  
  // 12-bit ADC (0-4095) mapped to 3.3V reference
  // Multiply by 2.0 to scale for 2:1 external divider
  float voltage = ((float)rawVal / 4095.0) * 3.3 * 2.0;
  return voltage;
}

// Compute discharge rate and accumulate capacity
void updateCapacityTest() {
  batteryVoltage = readBatteryVoltage();
  
  // Simulation voltage drift in MbedO if test is active
  if (testActive && batteryVoltage >= 4.2) {
    static float simVoltage = 4.2;
    simVoltage -= 0.0005; // Simulate slow discharge
    batteryVoltage = simVoltage;
  }
  
  unsigned long now = millis();
  float elapsedSec = (float)(now - lastSampleTime) / 1000.0;
  lastSampleTime = now;
  
  if (testActive) {
    // 1. Calculate active current: I = V / R
    dischargeCurrentMa = (batteryVoltage / LOAD_RESISTOR_OHM) * 1000.0;
    
    // 2. Accumulate Capacity (mAh): mAh += (mA * hours)
    accumulatedCapacityMah += (dischargeCurrentMa * elapsedSec) / 3600.0;
    
    // 3. Cutoff Safety Check
    if (batteryVoltage <= CUTOFF_VOLTAGE) {
      testActive = false;
      testCompleted = true;
      digitalWrite(LOAD_RELAY_PIN, LOW); // Open relay immediately (Stop discharge)
      Serial.println("[Battery Tester] CUTOFF REACHED! Test completed.");
      
      // Save final stats to SD card
      File file = SD.open(logFilepath, FILE_APPEND);
      if (file) {
        file.printf("COMPLETED,%.3f mAh\n", accumulatedCapacityMah);
        file.close();
      }
    }
    
    // 4. Periodically log stats to SD card CSV
    if (now - lastLogTime >= LOG_INTERVAL_MS) {
      lastLogTime = now;
      File file = SD.open(logFilepath, FILE_APPEND);
      if (file) {
        unsigned long durationS = (now - testStartTime) / 1000;
        file.printf("%lu,%.2f,%.1f,%.2f\n", durationS, batteryVoltage, dischargeCurrentMa, accumulatedCapacityMah);
        file.close();
        
        Serial.printf("[SD LOG] Time: %lu s | Volt: %.2f V | Current: %.1f mA | Capacity: %.2f mAh\n",
                      durationS, batteryVoltage, dischargeCurrentMa, accumulatedCapacityMah);
      }
    }
  } else {
    dischargeCurrentMa = 0.0;
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String stateStr = "READY";
  if (testActive) stateStr = "DISCHARGING";
  else if (testCompleted) stateStr = "COMPLETED";
  
  String json = "{\"voltage\":" + String(batteryVoltage, 2) + 
                 ",\"current\":" + String(dischargeCurrentMa, 1) + 
                 ",\"capacity\":" + String(accumulatedCapacityMah, 2) + 
                 ",\"state\":\"" + stateStr + "\"" +
                 ",\"sd\":\"" + sdStatus + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to control test execution
void handlePostControl() {
  if (server.hasArg("action")) {
    String action = server.arg("action");
    if (action == "start" && !testActive) {
      testActive = true;
      testCompleted = false;
      accumulatedCapacityMah = 0.0;
      testStartTime = millis();
      lastSampleTime = millis();
      lastLogTime = millis();
      digitalWrite(LOAD_RELAY_PIN, HIGH); // Close relay (Start discharge)
      
      // Initialize CSV log file on SD Card
      File file = SD.open(logFilepath, FILE_WRITE);
      if (file) {
        file.println("DurationS,VoltageV,CurrentMa,CapacityMah");
        file.close();
      }
      
      Serial.println("[Battery Tester] Discharge test started.");
    } 
    else if (action == "stop" && testActive) {
      testActive = false;
      digitalWrite(LOAD_RELAY_PIN, LOW); // Open relay (Stop discharge)
      Serial.println("[Battery Tester] Discharge test stopped by user.");
    }
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Battery Capacity Tester</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .capacity-val { font-size: 56px; font-weight: 800; font-family: monospace; color: #10b981; margin: 15px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #38bdf8; font-family: monospace; }\n";
  html += "  .status-badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; background-color: #334155; color: #94a3b8; }\n";
  html += "  .status-badge.active { background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .status-badge.done { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 20px; }\n";
  html += "  .btn { flex-grow: 1; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #065f46; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn-stop { background-color: #ef4444; }\n";
  html += "  .btn-stop:hover { background-color: #dc2626; }\n";
  html += "  .btn:hover { background-color: #047857; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Battery Capacity Tester</h1>\n";
  html += "  <div id=\"stateBadge\" class=\"status-badge\">READY</div>\n";
  html += "  <div id=\"capDisplay\" class=\"capacity-val\">0.0 mAh</div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Voltage</div><div class=\"metric-val\" id=\"voltDisplay\">0.00 V</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Current</div><div class=\"metric-val\" id=\"currDisplay\">0 mA</div></div>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn\" onclick=\"controlTest('start')\">START DISCHARGE</button>\n";
  html += "    <button class=\"btn btn-stop\" onclick=\"controlTest('stop')\">STOP TEST</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Discharge Resistor: 5.1 &Omega; | Cutoff Voltage: 3.0V | SD Status: <span id=\"sdStatus\">Disconnected</span></p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('capDisplay').innerText = data.capacity.toFixed(1) + ' mAh';\n";
  html += "        document.getElementById('voltDisplay').innerText = data.voltage.toFixed(2) + ' V';\n";
  html += "        document.getElementById('currDisplay').innerText = Math.round(data.current) + ' mA';\n";
  html += "        document.getElementById('sdStatus').innerText = data.sd;\n";
  
  html += "        const bdgEl = document.getElementById('stateBadge');\n";
  html += "        bdgEl.innerText = data.state;\n";
  html += "        if(data.state === 'DISCHARGING') {\n";
  html += "          bdgEl.className = 'status-badge active';\n";
  html += "        } else if(data.state === 'COMPLETED') {\n";
  html += "          bdgEl.className = 'status-badge done';\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function controlTest(actionVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('action', actionVal);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body })\n";
  html += "      .then(() => updateHUD());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BATTERY_PIN, INPUT);
  pinMode(LOAD_RELAY_PIN, OUTPUT);
  digitalWrite(LOAD_RELAY_PIN, LOW); // Load OFF initially
  
  // Initialize SD Card
  Serial.println("\nInitializing SD Card...");
  if (!SD.begin(SD_CS_PIN)) {
    sdStatus = "Mount Failed";
    Serial.println("[SD Error] Card Mount Failed!");
  } else {
    sdStatus = "Mounted";
    Serial.println("[SD] Mounted successfully.");
  }
  
  Serial.println("\nESP32 Battery Capacity Tester starting...");
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
}

void loop() {
  server.handleClient();
  
  // Continuously run tests and safety cutoff checks
  updateCapacityTest();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Relay**, **SD Card Module**, and **Potentiometer** (to simulate the battery cell) onto the canvas.
2. Wire Potentiometer wiper to **GPIO34**, Relay input to **GPIO12**, and SD CS to **GPIO5**.
3. Paste the code and click **Run**.
4. Set the simulated battery voltage potentiometer slider to 4.2V (maximum/high slider position).
5. Open your web browser and navigate to the printed IP address.
6. Click "START DISCHARGE". Verify that the relay closes (ON) and the capacity (mAh) begins accumulating on the webpage.
7. Slowly slide the simulated battery voltage slider down. When it drops below 3.0V, verify that the relay opens immediately, the test status changes to `COMPLETED`, and the final capacity value remains frozen on the screen.

## Expected Output
Serial Monitor:
```
[SD] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Battery Tester] Discharge test started.
[SD LOG] Time: 5 s | Volt: 4.10 V | Current: 803.9 mA | Capacity: 1.12 mAh
[SD LOG] Time: 10 s | Volt: 3.50 V | Current: 686.3 mA | Capacity: 2.08 mAh
[Battery Tester] CUTOFF REACHED! Test completed.
```

## Expected Canvas Behavior
* Clicking the webpage buttons toggles the state of the simulated relay widget.
* Draining the simulated battery slider triggers automatic relay cutoff.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `readBatteryVoltage()` | Computes actual cell voltage using ADC ratios. |
| `batteryVoltage <= CUTOFF_VOLTAGE` | Safety check that prevents discharging the lithium cell below the minimum threshold. |
| `accumulatedCapacityMah += ...` | Integrates active current draw over elapsed time to calculate capacity. |
| `SD.open(..., FILE_APPEND)` | Appends discharge curves (Time, Voltage, Current, mAh) to the SD Card CSV. |

## Hardware & Safety Concept: Over-Discharge Protection and Thermal Dissipation
* **Over-Discharge Protection**: Discharging a Lithium-Ion battery below 2.5V damages its internal structure, reducing its capacity and potentially creating a fire hazard during subsequent recharges. Always implement a hardware or software **cutoff limit** (typically 3.0V) to isolate the load when the battery is drained.
* **Thermal Dissipation**: Discharging an 18650 cell at 1A through a 5.1 Ω load resistor generates approximately 4 Watts of heat (`P = V² / R`). This resistor will become extremely hot. Use a high-power wirewound aluminum resistor rated for at least 10 Watts, and keep it away from plastic casings and logic wires.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a battery icon filled proportionally to the voltage.
2. **Alert buzzer**: Add a buzzer (GPIO 15) to play a melody when the test is completed.
3. **True Current Sensor**: Connect an ACS712 current sensor (Project 138) to measure actual discharge current rather than calculating it mathematically.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Voltage readings are incorrect | Divider ratio wrong | Verify that the resistors in the voltage divider are equal (e.g. both 10 kΩ) to create a 2:1 division ratio |
| Relay does not actuate | Solenoid power missing | Check that your relay board is powered from the 5V Vin rail, as many relays will not actuate at 3.3V |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [138 - ESP32 SD Card Energy Logger Web Status](138-esp32-sd-card-energy-logger-web-status.md)
- [160 - Ultrasonic Wind Anemometer IoT Dashboard](160-esp32-ultrasonic-wind-anemometer-iot-dashboard.md) (Next project)
