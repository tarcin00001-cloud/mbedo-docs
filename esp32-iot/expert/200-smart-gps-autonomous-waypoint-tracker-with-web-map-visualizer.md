# 200 - Smart GPS Autonomous Waypoint Tracker with Web Map visualizer

Build an autonomous location logger and map visualizer on the ESP32 that reads NMEA sentences from a NEO-6M GPS module over UART, hosts a local web server, and renders your real-time path trail using the Leaflet.js mapping engine.

## Goal
Learn how to parse NMEA strings using the TinyGPS++ library, manage Hardware Serial UART connections, serve mapping scripts, update location indicators, and build web maps.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A NEO-6M GPS module is wired to hardware serial UART pins (GPIO 16, 17). The ESP32 parses incoming GPS satellite signals. It runs an HTTP server on port 80. Navigating to the ESP32's IP address displays a mapping interface. The page loads an interactive Leaflet.js map, polls the ESP32 for coordinate values, and plots your current location marker and breadcrumb route trail in real-time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NEO-6M GPS Module | `gps` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M GPS | TXD | GPIO16 | Yellow | Hardware Serial2 RX |
| NEO-6M GPS | RXD | GPIO17 | Green | Hardware Serial2 TX (uses voltage divider) |
| LED | Anode (+) | GPIO12 | Red | GPS lock indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The NEO-6M GPS antenna works best with a direct line of sight to the sky. Testing indoors may result in a delay in obtaining a satellite lock.

## Code
```cpp
// GPS Waypoint Tracker (Hardware Serial2 UART + TinyGPS++ + Web Server + Leaflet.js Maps)
#include <WiFi.h>
#include <WebServer.h>
#include <TinyGPS++.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// NEO-6M GPS UART Pins (Serial2)
#define RXD2 16
#define TXD2 17

TinyGPSPlus gps;
WebServer server(80);

const int LOCK_LED = 12;

// Coordinate cache (Default to London if no lock exists)
double latitudeVal = 51.5074;
double longitudeVal = -0.1278;
float speedKmph = 0.0;
int satCount = 0;
bool gpsHasLock = false;

// Serve root webpage containing Leaflet.js map container
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>GPS Waypoint HUD</title>\n";
  // Leaflet.js CDN CSS stylesheet
  html += "<link rel=\"stylesheet\" href=\"https://unpkg.com/leaflet@1.9.4/dist/leaflet.css\" />\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 720px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  #map { width: 100%; height: 350px; border-radius: 8px; border: 1px solid #334155; margin: 20px 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 12px; margin-bottom: 20px; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 10px; border: 1px solid #334155; text-align: center; }\n";
  html += "  .metric-label { font-size: 10px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 16px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>NEO-6M GPS Trail HUD</h1>\n";
  
  // Mapping placeholder
  html += "  <div id=\"map\"></div>\n";
  
  // Metrics Grid
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Position</div><div class=\"metric-val\" id=\"posVal\">51.507, -0.127</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Speed (KMPH)</div><div class=\"metric-val\" id=\"speedVal\">0.0</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Satellites</div><div class=\"metric-val\" id=\"satVal\">0</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Leaflet Maps | ESP32 NEO-6M UART Gateway</p>\n";
  html += "</div>\n";
  
  // Leaflet.js CDN Map Engine Script
  html += "<script src=\"https://unpkg.com/leaflet@1.9.4/dist/leaflet.js\"></script>\n";
  html += "<script>\n";
  // Initialize Leaflet Map centered on current location
  html += "  let map = L.map('map').setView([" + String(latitudeVal, 6) + ", " + String(longitudeVal, 6) + "], 15);\n";
  // Load free OSM tile layers
  html += "  L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {\n";
  html += "    attribution: '&copy; OpenStreetMap contributors'\n";
  html += "  }).addTo(map);\n";
  
  html += "  let marker = L.marker([" + String(latitudeVal, 6) + ", " + String(longitudeVal, 6) + "]).addTo(map);\n";
  html += "  let pathLine = L.polyline([], {color: '#38bdf8'}).addTo(map);\n";
  
  // AJAX Polling updates
  html += "  function updateGPS() {\n";
  html += "    fetch('/api/gps')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('posVal').innerText = data.lat.toFixed(4) + ', ' + data.lng.toFixed(4);\n";
  html += "        document.getElementById('speedVal').innerText = data.speed.toFixed(1);\n";
  html += "        document.getElementById('satVal').innerText = data.sats;\n";
  
  html += "        if (data.lock) {\n";
  html += "          const newLatLng = [data.lat, data.lng];\n";
  html += "          marker.setLatLng(newLatLng);\n";
  // Add breadcrumb trail coordinate points
  html += "          pathLine.addLatLng(newLatLng);\n";
  html += "          map.panTo(newLatLng);\n";
  html += "        }\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    setInterval(updateGPS, 2000); // Poll GPS metrics every 2 seconds\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST route returning active coordinates
void handleGetGps() {
  String json = "{\"lat\":" + String(latitudeVal, 6) + 
                 ",\"lng\":" + String(longitudeVal, 6) + 
                 ",\"speed\":" + String(speedKmph, 2) + 
                 ",\"sats\":" + String(satCount) + 
                 ",\"lock\":" + String(gpsHasLock ? "true" : "false") + "}";
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LOCK_LED, OUTPUT);
  digitalWrite(LOCK_LED, LOW);
  
  // 1. Initialize GPS Hardware Serial 2 (Baud Rate 9600 for NEO-6M)
  Serial2.begin(9600, SERIAL_8N1, RXD2, TXD2);
  Serial.println("[GPS] Serial2 UART started (9600 Baud).");
  
  Serial.println("\nESP32 GPS Map Tracker starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Start server
  server.on("/", HTTP_GET, handleRoot);
  server.on("/api/gps", HTTP_GET, handleGetGps);
  server.begin();
  
  Serial.println("HTTP Mapping Server active.");
}

void loop() {
  server.handleClient();
  
  // 2. Read and parse incoming NEO-6M NMEA sentences
  while (Serial2.available() > 0) {
    if (gps.encode(Serial2.read())) {
      // If a valid location update is parsed
      if (gps.location.isValid()) {
        latitudeVal = gps.location.lat();
        longitudeVal = gps.location.lng();
        gpsHasLock = true;
        
        digitalWrite(LOCK_LED, HIGH); // Solid LED indicates satellite lock
        
        if (gps.speed.isValid()) {
          speedKmph = gps.speed.kmph();
        }
        if (gps.satellites.isValid()) {
          satCount = gps.satellites.value();
        }
      }
    }
  }
  
  // Mock simulation coordinates in workspace if GPS does not have lock
  if (!gpsHasLock) {
    static float mockLat = 51.5074;
    static float mockLng = -0.1278;
    // Add small drift to simulate a walking path
    mockLat += 0.0001 * (random(-1, 2));
    mockLng += 0.0001 * (random(-1, 2));
    latitudeVal = mockLat;
    longitudeVal = mockLng;
    speedKmph = 4.2;
    satCount = 8;
    gpsHasLock = true;
    digitalWrite(LOCK_LED, HIGH);
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **NEO-6M GPS**, and **LED** onto the canvas.
2. Wire GPS TX/RX to **GPIO16** and **GPIO17**, and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Verify that the console prints the connection log and launches the server.
5. In simulation, since a satellite signal cannot be read directly, mock NMEA sentences by typing coordinates in the serial terminal: `SYS:GPS:51.5074,-0.1278` (latitude, longitude).
6. Verify that the LED turns ON (simulating a lock).
7. Open your web browser, navigate to the IP address, and verify that the Leaflet map loads, centered on London, showing the active location marker and trail line.

## Expected Output
Serial Monitor:
```
[GPS] Serial2 UART started (9600 Baud).
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP Mapping Server active.
[GPS Lock] Satellites found: 8 | Coord: 51.5074, -0.1278
```

## Expected Canvas Behavior
* Booting the system launches the NMEA parsing loops. Mocking coordinates updates the map marker and draws path trails on the web client interface.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `gps.encode(Serial2.read())` | Feeds raw NMEA characters into the TinyGPS parser object. |
| `gps.location.isValid()` | Returns true if the NEO-6M has decoded latitude/longitude. |
| `L.tileLayer(...)` | Leaflet command loading OpenStreetMap image layers. |

## Hardware & Safety Concept: GPS Cold Starts and Power Consumption
* **GPS Cold Starts**: When first powered ON, a GPS module has no cached satellite data. Resolving its initial position (cold start) can take 5–10 minutes depending on weather conditions. Once locked, hot starts (powering OFF and back ON quickly) take under 5–10 seconds because the GPS caches satellite data.
* **Power Consumption**: NEO-6M GPS modules pull up to 80–100 mA when searching for satellites. Do not power the module from the ESP32's 3.3V out pin; connect it to the 5V rail (NEO-6M has an onboard regulator) to prevent logic level voltage sag.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current latitude, longitude, and satellite count.
2. **SD Card Route logger**: Modify the code to save all logged path coordinates to a CSV file on an SD Card (Project 136).
3. **Azure Telemetry logger**: Integrate this project with Project 185 to upload live GPS coordinates to Azure IoT Hub.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Satellite lock never completes | Indoor signal block | GPS signals cannot penetrate concrete walls or metal roofs. Place the antenna outside or near a window |
| Latitude/Longitude shows 0.0000 | NMEA parsing failed | Ensure that the GPS TX pin connects to the ESP32 RX2 pin (GPIO 16) and that the baud rate matches 9600 in code |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [185 - Azure IoT Hub direct telemetry logger](185-azure-iot-hub-direct-telemetry-logger.md)
- [198 - Bluetooth & WiFi Robot HUD Console](198-bluetooth-wifi-robot-hud-console.md)
