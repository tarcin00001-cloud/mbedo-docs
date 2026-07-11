# 98 - Gas Leakage Alarm IoT Node (MQ-2 + Buzzer + Relay + Web Alert)

Build an integrated safety station on the ESP32 that samples an analog MQ-2 gas sensor on GPIO 34, triggers a local audio alarm on a passive buzzer (GPIO 15) and activates a ventilation relay (GPIO 13) when gas levels exceed a safety threshold, and hosts a web dashboard displaying real-time safety metrics.

## Goal
Learn how to read analog gas sensors, integrate multiple warning outputs, write safety threshold check loops, design web dashboards, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A gas sensor is on GPIO 34, a buzzer on GPIO 15, and a relay on GPIO 13. The ESP32 hosts a web page displaying gas concentration percentages and warning states. If gas concentration exceeds 40%, the buzzer sounds an alarm, the relay closes (to turn on an exhaust fan), and the web dashboard displays a flashing warning alert.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas Sensor Module | `potentiometer` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

> **MbedO Note:** In simulation, use a potentiometer to simulate the analog output of the MQ-2 gas sensor.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Gas Sensor | AO (Analog Out) | GPIO34 | Yellow | Gas density signal |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Warning buzzer |
| Relay Module | IN (Signal) | GPIO13 | Orange | Exhaust fan relay |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the buzzer to GPIO 15 and the relay to GPIO 13. Power the gas sensor and relay from the 5V Vin rail.

## Code
```cpp
// Gas Leakage Alarm IoT Node (MQ-2 + Buzzer + Relay + Web Alert)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int GAS_PIN = 34;
const int BUZZER_PIN = 15;
const int RELAY_PIN = 13;

WebServer server(80);

int gasPercent = 0;
const int ALARM_THRESHOLD = 40; // Alarm triggers above 40%

// Get active safety status string
String getStatusString() {
  return (gasPercent > ALARM_THRESHOLD) ? "DANGER: LEAK DETECTED" : "SAFE";
}

// HTTP API endpoint returning JSON data
void handleGetGasData() {
  bool relayState = (digitalRead(RELAY_PIN) == HIGH);
  String json = "{\"percent\":" + String(gasPercent) + 
                 ",\"status\":\"" + getStatusString() + "\"" +
                 ",\"relay\":" + String(relayState ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Gas Safety Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .value { font-size: 64px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .value.alarm { color: #ef4444; animation: blink 1s infinite; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin-bottom: 20px; }\n";
  html += "  .badge.safe { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .badge.alarm { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  .fan-status { font-size: 16px; color: #94a3b8; margin-top: 15px; }\n";
  html += "  .fan-status.active { color: #38bdf8; font-weight: bold; }\n";
  html += "  @keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Gas Safety Console</h1>\n";
  html += "  <div id=\"gasBadge\" class=\"badge safe\">SAFE</div>\n";
  html += "  <div id=\"gasDisplay\" class=\"value\">0%</div>\n";
  html += "  <div id=\"fanDisplay\" class=\"fan-status\">Exhaust Fan: Off</div>\n";
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateSafetyData() {\n";
  html += "    fetch('/api/gas')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('gasDisplay').innerText = data.percent + '%';\n";
  html += "        const valEl = document.getElementById('gasDisplay');\n";
  html += "        const bdgEl = document.getElementById('gasBadge');\n";
  html += "        const fanEl = document.getElementById('fanDisplay');\n";
  
  html += "        bdgEl.innerText = data.status;\n";
  
  html += "        if (data.percent > " + String(ALARM_THRESHOLD) + ") {\n";
  html += "          valEl.classList.add('alarm');\n";
  html += "          bdgEl.className = 'badge alarm';\n";
  html += "          fanEl.innerText = 'Exhaust Fan: ACTIVE';\n";
  html += "          fanEl.className = 'fan-status active';\n";
  html += "        } else {\n";
  html += "          valEl.classList.remove('alarm');\n";
  html += "          bdgEl.className = 'badge safe';\n";
  html += "          fanEl.innerText = 'Exhaust Fan: Off';\n";
  html += "          fanEl.className = 'fan-status';\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateSafetyData();\n";
  html += "    setInterval(updateSafetyData, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(GAS_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  
  Serial.println("\nESP32 Gas Alarm Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/gas", handleGetGasData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Read analog gas sensor
  int rawVal = analogRead(GAS_PIN);
  gasPercent = map(rawVal, 0, 4095, 0, 100);
  
  // Safety checking loop
  if (gasPercent > ALARM_THRESHOLD) {
    digitalWrite(RELAY_PIN, HIGH); // Turn on ventilation fan
    
    // Generate rapid pulsing buzzer alarm (siren sound)
    static unsigned long lastTone = 0;
    if (millis() - lastTone >= 250) {
      static bool soundOn = false;
      soundOn = !soundOn;
      if (soundOn) {
        tone(BUZZER_PIN, 1500);
      } else {
        noTone(BUZZER_PIN);
      }
      lastTone = millis();
    }
  } else {
    digitalWrite(RELAY_PIN, LOW); // Turn off ventilation fan
    noTone(BUZZER_PIN);          // Silence buzzer
  }
  
  delay(2);
}
```,Description:
