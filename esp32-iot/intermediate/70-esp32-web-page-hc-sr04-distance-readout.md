# 70 - Web Page HC-SR04 distance readout

Build a web-connected range finder on the ESP32 that hosts a control page, reads distance metrics from an HC-SR04 ultrasonic sensor (Trigger on GPIO 12, Echo on GPIO 14), serves a JSON API endpoint, and uses client-side JavaScript (AJAX polling) to refresh the distance readout every second.

## Goal
Learn how to actuate ultrasonic sensors, measure return echo pulse durations, calculate distances in centimeters, implement client-side AJAX polling loops, and format JSON telemetry.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An HC-SR04 ultrasonic sensor is connected to GPIO 12 (Trigger) and GPIO 14 (Echo). Navigating to the ESP32's IP address displays a webpage showing the distance in centimeters. JavaScript on the page polls `/api/distance` every second, updating the display value immediately.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 Sensor | TRIG | GPIO12 | Yellow | Trigger pulse input |
| HC-SR04 Sensor | ECHO | GPIO14 | Green | Echo pulse output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Sensor power rails |

> **Wiring tip:** Power the HC-SR04 sensor from the 5V Vin rail.

## Code
```cpp
// Web Page HC-SR04 distance readout (Asynchronous range finder dashboard)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int TRIG_PIN = 12;
const int ECHO_PIN = 14;

WebServer server(80);

// Helper function to read distance from HC-SR04 sensor
float readDistanceCm() {
  // 1. Trigger sensor with a 10 microsecond HIGH pulse
  digitalWrite(TRIG_PIN, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  
  // 2. Measure return echo pulse duration in microseconds
  // Timeout set to 26000 us (approx 4.5 meters max range)
  long duration = pulseIn(ECHO_PIN, HIGH, 26000);
  
  if (duration == 0) {
    return -1.0; // Out of range or connection error
  }
  
  // 3. Calculate distance in centimeters
  // Speed of sound: 343 m/s -> 0.0343 cm/us
  // Distance = (Duration * Speed of Sound) / 2 (round trip)
  float distanceCm = (float)duration * 0.0343 / 2.0;
  return distanceCm;
}

// Root URL Route Handler ("/")
// Serves the HTML dashboard containing CSS, the gauge, and the AJAX polling loop
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Ultrasonic Range Finder</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  
  html += "  .metric { display: flex; justify-content: center; align-items: baseline; margin: 30px 0; }\n";
  html += "  .value { color: #f1f5f9; font-family: monospace; font-size: 56px; font-weight: bold; line-height: 1; }\n";
  html += "  .unit { font-size: 20px; color: #94a3b8; margin-left: 8px; }\n";
  
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Ultrasonic Range Finder</h1>\n";
  
  html += "  <div class=\"metric\">\n";
  html += "    <span class=\"value\" id=\"distVal\">--.-</span>\n";
  html += "    <span class=\"unit\">cm</span>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // Client-side AJAX polling script using Fetch API
  html += "<script>\n";
  html += "  function updateDistance() {\n";
  html += "    fetch('/api/distance')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const valEl = document.getElementById('distVal');\n";
  html += "        if (data.distance >= 0) {\n";
  html += "          valEl.innerText = data.distance.toFixed(1);\n";
  html += "          valEl.style.color = '#f1f5f9';\n";
  html += "        } else {\n";
  html += "          valEl.innerText = 'Error';\n";
  html += "          valEl.style.color = '#ef4444';\n";
  html += "        }\n";
  html += "      })\n";
  html += "      .catch(err => console.error('Error fetching distance data:', err));\n";
  html += "  }\n";
  
  // Start polling loop once page finishes loading
  html += "  window.onload = function() {\n";
  html += "    updateDistance();\n";
  html += "    setInterval(updateDistance, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// API Telemetry URL Handler ("/api/distance")
void handleDistanceAPI() {
  float distance = readDistanceCm();
  
  // Compile JSON telemetry string
  String json = "{\"distance\":" + String(distance, 1) + "}";
  
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  
  digitalWrite(TRIG_PIN, LOW);
  
  Serial.println("\nESP32 Range Finder Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/api/distance", handleDistanceAPI);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **HC-SR04** onto the canvas.
2. Wire TRIG to **GPIO12** and ECHO to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the HC-SR04 distance slider. Watch the value update on the webpage.

## Expected Output
Serial Monitor:
```
ESP32 Range Finder Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
```

Browser Console (Inspect Network tab):
* Continuous lightweight GET requests to `/api/distance` returning JSON strings (`{"distance":120.5}`).

## Expected Canvas Behavior
* Adjusting the HC-SR04 distance slider updates the value displayed on the webpage in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pulseIn(ECHO_PIN, HIGH, 26000)` | Measures the return echo pulse duration in microseconds with a 26 ms timeout. |
| `duration * 0.0343 / 2.0` | Converts pulse duration to distance in centimeters. |
| `setInterval(updateDistance, 1000)` | Registers a browser timer to call the update function every 1000 ms. |
| `server.send(200, "application/json", ...)` | Serves the distance payload with the correct JSON MIME type. |

## Hardware & Safety Concept: Sensor Timeouts and Non-Blocking Servers
The `pulseIn()` function blocks execution while waiting for the echo pin to go HIGH and then LOW. If the sensor is disconnected or out of range, `pulseIn()` can block for up to 1 second, causing the web server to freeze. Always configure a **timeout limit** (e.g. 26 ms) to abort the measurement and keep the server responsive if a packet is lost.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current distance.
2. **Proximity Alert Buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the distance drops below 15 cm.
3. **SPIFFS integration**: Store the webpage assets in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |
| Distance always displays "Error" | Echo pin timeout | Confirm that Echo and Trigger pins are wired correctly (GPIO 14 and 12) |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [61 - ESP32 Web Server AJAX Telemetry Auto-refresh](61-esp32-web-server-ajax-telemetry-auto-refresh.md)
- [69 - ESP32 Web Page LDR light sensor bar display](69-esp32-web-page-ldr-light-sensor-bar-display.md)
- [73 - ESP32 WebSocket Sensor Streamer](73-esp32-websocket-sensor-streamer.md)
