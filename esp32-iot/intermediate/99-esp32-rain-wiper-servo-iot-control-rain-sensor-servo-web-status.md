# 99 - Rain Wiper Servo IoT Control (Rain sensor + Servo + Web Status)

Build a web-connected automated weather accessory station on the ESP32 that samples an analog rain sensor on GPIO 34, drives a servo motor windshield wiper simulator on GPIO 13 using a non-blocking sweep state machine, and hosts a web dashboard displaying real-time precipitation metrics.

## Goal
Learn how to sample analog rain sensors, implement non-blocking servo sweep routines, serve interactive web dashboards, and package telemetry data into JSON endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A rain sensor is on GPIO 34, and a servo motor on GPIO 13. The ESP32 hosts a web page displaying precipitation percentages and wiper states. If moisture levels exceed 30%, the servo begins sweeping back and forth automatically, and the web page displays a warning status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Rain Sensor Module | `potentiometer` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the rain sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rain Sensor | AO (Analog Out) | GPIO34 | Yellow | Rain intensity signal |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Wiper motor control |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the servo and rain sensor from the 5V Vin rail.

## Code
```cpp
// Rain Wiper Servo IoT Control (Precipitation telemetry + automated wiper sweeps)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RAIN_PIN = 34;
const int SERVO_PIN = 13;

Servo wiperServo;
WebServer server(80);

int rainPercent = 0;
bool wipersActive = false;

// Non-blocking servo sweep variables
unsigned long lastWiperStep = 0;
const unsigned long WIPER_STEP_DELAY = 15; // Sweep speed delay
int servoAngle = 0;
int sweepDirection = 1; // 1 for forward, -1 for backward

// Get active weather status string
String getWeatherStatus() {
  if (rainPercent < 15) return "Dry / Clear";
  if (rainPercent >= 15 && rainPercent < 50) return "Light Rain / Drizzle";
  return "HEAVY PRECIPITATION";
}

// HTTP API endpoint returning JSON data
void handleGetRainData() {
  String json = "{\"rain\":" + String(rainPercent) + 
                 ",\"status\":\"" + getWeatherStatus() + "\"" +
                 ",\"wipers\":" + String(wipersActive ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Wiper Controller Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .value { font-size: 64px; font-weight: 800; font-family: monospace; color: #38bdf8; margin: 20px 0; }\n";
  html += "  .value.active { color: #f59e0b; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .badge.dry { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.wet { background-color: #92400e; color: #fef3c7; }\n";
  html += "  .badge.heavy { background-color: #991b1b; color: #fee2e2; animation: flash 1s infinite; }\n";
  html += "  .wiper-status { font-size: 16px; color: #94a3b8; margin-top: 15px; }\n";
  html += "  .wiper-status.active { color: #10b981; font-weight: bold; }\n";
  html += "  @keyframes flash { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Wiper Controller</h1>\n";
  html += "  <div id=\"weatherBadge\" class=\"badge dry\">DRY</div>\n";
  html += "  <div id=\"rainDisplay\" class=\"value\">0%</div>\n";
  html += "  <div id=\"wiperDisplay\" class=\"wiper-status\">Wiper: Off</div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateRainData() {\n";
  html += "    fetch('/api/rain')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('rainDisplay').innerText = data.rain + '%';\n";
  html += "        const valEl = document.getElementById('rainDisplay');\n";
  html += "        const bdgEl = document.getElementById('weatherBadge');\n";
  html += "        const wipEl = document.getElementById('wiperDisplay');\n";
  
  html += "        bdgEl.innerText = data.status;\n";
  
  if (html += "        if (data.rain > 30) {\n") {
    // Wet / Active Wipers
  }
  html += "        if (data.rain < 15) {\n";
  html += "          bdgEl.className = 'badge dry';\n";
  html += "          valEl.classList.remove('active');\n";
  html += "        } else if (data.rain >= 15 && data.rain < 50) {\n";
  html += "          bdgEl.className = 'badge wet';\n";
  html += "          valEl.classList.add('active');\n";
  html += "        } else {\n";
  html += "          bdgEl.className = 'badge heavy';\n";
  html += "          valEl.classList.add('active');\n";
  html += "        }\n";
  
  html += "        if (data.wipers === 1) {\n";
  html += "          wipEl.innerText = 'Wiper Mode: SWEEPING';\n";
  html += "          wipEl.className = 'wiper-status active';\n";
  html += "        } else {\n";
  html += "          wipEl.innerText = 'Wiper Mode: Off';\n";
  html += "          wipEl.className = 'wiper-status';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateRainData();\n";
  html += "    setInterval(updateRainData, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RAIN_PIN, INPUT);
  wiperServo.attach(SERVO_PIN);
  wiperServo.write(0); // Park wiper
  
  Serial.println("\nESP32 Weather Wiper Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/rain", handleGetRainData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read rain sensor
  // Rain sensors output lower voltages when wet (0V at maximum water, 3.3V when dry)
  // Map value to humidity index: 100% is wet, 0% is dry
  int rawVal = analogRead(RAIN_PIN);
  rainPercent = map(rawVal, 4095, 0, 0, 100);
  if (rainPercent < 0) rainPercent = 0;
  if (rainPercent > 100) rainPercent = 100;
  
  // Wiper trigger state check
  wipersActive = (rainPercent > 30);
  
  if (wipersActive) {
    // Non-blocking servo sweep scheduler
    unsigned long now = millis();
    if (now - lastWiperStep >= WIPER_STEP_DELAY) {
      servoAngle += sweepDirection;
      
      // Toggle sweep direction at limit bounds
      if (servoAngle >= 180) {
        servoAngle = 180;
        sweepDirection = -1;
      } else if (servoAngle <= 0) {
        servoAngle = 0;
        sweepDirection = 1;
      }
      
      wiperServo.write(servoAngle);
      lastWiperStep = now;
    }
  } else {
    // Slowly return wiper to park position if dry
    if (servoAngle > 0) {
      unsigned long now = millis();
      if (now - lastWiperStep >= WIPER_STEP_DELAY) {
        servoAngle--;
        wiperServo.write(servoAngle);
        lastWiperStep = now;
      }
    }
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the rain sensor), and **Servo** onto the canvas.
2. Wire Potentiometer to **GPIO34** and Servo to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer slider.
5. Move the potentiometer slider to simulate rain (lower the value to simulate water density). Verify that the servo sweeps back and forth, and the webpage updates immediately.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/rain`):
```json
{"rain":42,"status":"Light Rain / Drizzle","wipers":1}
```

## Expected Canvas Behavior
* Adjusting the potentiometer simulates rain levels. When the level exceeds 30%, the servo widget sweeps between 0° and 180° continuously.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `map(rawVal, 4095, 0, 0, 100)` | Inverts and maps the rain sensor output (0V is wet, 3.3V is dry). |
| `now - lastWiperStep >= 15` | Schedules a non-blocking 15 ms step update for the wiper sweep. |
| `sweepDirection = -1` | Inverts the servo sweep direction when it reaches the 180° limit. |
| `wiperServo.write(servoAngle)` | Drives the servo to the current calculated sweep angle. |

## Hardware & Safety Concept: Non-Blocking Sweeps vs Server Freeze
Using a blocking sweep (e.g. using a `for (int i=0; i<=180; i++) { servo.write(i); delay(15); }` loop) halts all other code execution on the ESP32 for several seconds. During this time, the web server cannot respond to incoming client requests, resulting in dropped connections. Enforcing **asynchronous step sweeps** keeps the server responsive.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a warning screen when the wipers are active.
2. **Dynamic Wiper Speeds**: Modify the code to sweep the servo faster if the rain intensity exceeds 70%.
3. **Buzzer alert tone**: Sound a brief musical warning tone on a buzzer (GPIO 15) when the wipers first activate.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not sweep | Power rails missing | Confirm that the servo is powered from the 5V Vin rail, not the 3.3V rail |
| Webpage updates but values do not change | Potentiometer wired wrong | Check that the wiper pin is connected to GPIO 34 |
| Web page responds slowly | Blocking loops | Ensure that there are no blocking delay functions inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [98 - ESP32 Gas Leakage Alarm IoT Node](98-esp32-gas-leakage-alarm-iot-node-mq-2-buzzer-relay-web-alert.md)
- [100 - ESP32 Automatic Smart Fan IoT Console](100-esp32-automatic-smart-fan-iot-console.md) (Next project)
