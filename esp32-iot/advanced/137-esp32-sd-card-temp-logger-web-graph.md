# 137 - SD Card Temp Logger Web Graph (Read CSV -> HTML5 chart)

Build an automated environmental data logger on the ESP32 that samples a DHT22 sensor on GPIO 14, appends temperature and humidity readings to a local `/temp_log.csv` file on an SD card via SPI, and hosts a web dashboard displaying a scrolling HTML5 Canvas line chart parsed directly from the CSV.

## Goal
Learn how to sample DHT22 sensors, mount and append data to CSV files in SPI SD cards, stream file contents over HTTP, and parse CSV strings using client-side JavaScript to render HTML5 Canvas charts.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A DHT22 sensor is on GPIO 14, and an SD card module on SPI. Every 10 seconds, the ESP32 reads the temperature and humidity, logs them with a timestamp to `/temp_log.csv` on the SD card, and prints values to the Serial Monitor. Navigating to the ESP32's IP address displays a webpage that fetches the raw CSV, parses it, and plots the temperature history on a line graph.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Micro SD Card (FAT16/FAT32 formatted) | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Temperature & humidity input |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK (Clock) | GPIO18 | Green | SPI clock output |
| SD Card Module | MISO (Data Out)| GPIO19 | Blue | SPI master input |
| SD Card Module | MOSI (Data In) | GPIO23 | White | SPI master output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 10 kΩ pull-up resistor between the DHT22 data pin and the 3.3V rail. Power both modules from the 5V Vin rail.

## Code
```cpp
// SD Card Temp Logger Web Graph (DHT22 Logging + SPI SD append + CSV Chart Parser)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <SD.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int SD_CS_PIN = 5;
const char* logFilepath = "/temp_log.csv";

WebServer server(80);

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log data every 10 seconds

// HTTP Handler to stream raw CSV data
void handleGetCSV() {
  if (SD.exists(logFilepath)) {
    File f = SD.open(logFilepath, FILE_READ);
    server.streamFile(f, "text/csv");
    f.close();
  } else {
    server.send(404, "text/plain", "Log File Not Found");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Climate Logger Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  canvas { background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 20px; width: 100%; aspect-ratio: 2/1; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; background-color: #065f46; color: #a7f3d0; margin-bottom: 15px; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>SD Environmental Logger</h1>\n";
  html += "  <div class=\"badge\">SD LOGGING SYSTEM ACTIVE</div>\n";
  
  // Real-time canvas
  html += "  <canvas id=\"chartCanvas\" width=\"400\" height=\"200\"></canvas>\n";
  
  html += "  <p class=\"footer\">Logging interval: 10 seconds | Graph parses raw CSV logs</p>\n";
  html += "</div>\n";
  
  // JavaScript CSV parsing and drawing script
  html += "<script>\n";
  html += "  const canvas = document.getElementById('chartCanvas');\n";
  html += "  const ctx = canvas.getContext('2d');\n";
  
  html += "  function drawChart(temps) {\n";
  html += "    ctx.clearRect(0, 0, canvas.width, canvas.height);\n";
  
  // Grid Lines
  html += "    ctx.strokeStyle = '#334155';\n";
  html += "    ctx.lineWidth = 1;\n";
  html += "    for(let y = 50; y < 200; y += 50) {\n";
  html += "      ctx.beginPath();\n";
  html += "      ctx.moveTo(0, y);\n";
  html += "      ctx.lineTo(canvas.width, y);\n";
  html += "      ctx.stroke();\n";
  html += "    }\n";
  
  // Draw Data Path line
  html += "    if (temps.length < 2) return;\n";
  html += "    ctx.strokeStyle = '#38bdf8';\n";
  html += "    ctx.lineWidth = 3;\n";
  html += "    ctx.beginPath();\n";
  
  html += "    const step = canvas.width / (temps.length - 1);\n";
  html += "    const minT = Math.min(...temps) - 2;\n";
  html += "    const maxT = Math.max(...temps) + 2;\n";
  html += "    const range = maxT - minT;\n";
  
  html += "    temps.forEach((val, i) => {\n";
  html += "      const x = i * step;\n";
  html += "      const y = 200 - (((val - minT) / range) * 160 + 20); // Scaled to fit canvas\n";
  html += "      if (i === 0) ctx.moveTo(x, y);\n";
  html += "      else ctx.lineTo(x, y);\n";
  html += "    });\n";
  html += "    ctx.stroke();\n";
  html += "  }\n";
  
  // Fetch and parse raw CSV
  html += "  function loadCSV() {\n";
  html += "    fetch('/log.csv')\n";
  html += "      .then(response => response.text())\n";
  html += "      .then(text => {\n";
  html += "        const lines = text.trim().split('\\n');\n";
  html += "        const temps = [];\n";
  
  // Skip header line
  html += "        for (let i = 1; i < lines.length; i++) {\n";
  html += "          const cols = lines[i].split(',');\n";
  html += "          if (cols.length >= 3) {\n";
  html += "            temps.push(parseFloat(cols[1])); // Column 1: TempC\n";
  html += "          }\n";
  html += "        }\n";
  
  // Limit graph to last 20 readings
  html += "        const lastReadings = temps.slice(-20);\n";
  html += "        drawChart(lastReadings);\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    loadCSV();\n";
  html += "    setInterval(loadCSV, 5000); // Fetch CSV every 5 seconds\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  Serial.println("\nInitializing SD Card...");
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("[SD Error] Card Mount Failed! System halted.");
    for (;;);
  }
  Serial.println("[SD] Mounted successfully.");
  
  // Initialize the CSV file with headers if it doesn't exist
  if (!SD.exists(logFilepath)) {
    File f = SD.open(logFilepath, FILE_WRITE);
    if (f) {
      f.println("Timestamp,TempC,Humidity");
      f.close();
      Serial.println("[SD] Initialized new CSV file.");
    }
  }
  
  Serial.println("\nESP32 Climate Logger starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/log.csv", handleGetCSV);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // Log climate values to SD card periodically
  unsigned long now = millis();
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    lastLogTime = now;
    
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    
    if (!isnan(t) && !isnan(h)) {
      // Open file in append mode to add data without overwriting
      File file = SD.open(logFilepath, FILE_APPEND);
      if (file) {
        unsigned long timestampS = now / 1000;
        file.printf("%lu,%.1f,%.1f\n", timestampS, t, h);
        file.close();
        
        Serial.printf("[SD LOG] Saved data -> Time: %lu, Temp: %.1f C, Humid: %.1f%%\n", 
                      timestampS, t, h);
      } else {
        Serial.println("[SD Error] Failed to open file for appending!");
      }
    } else {
      Serial.println("[DHT Error] Failed to read sensor data!");
    }
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **SD Card Module** onto the canvas.
2. Wire DHT22 signal to **GPIO14**, and SD CS to **GPIO5** (standard SCK/MISO/MOSI to GPIO18/19/23).
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Change the temperature slider on the simulated DHT22 widget (e.g. set it to 28 °C).
6. Wait 15 seconds. Verify that the line graph on the webpage plots the temperature change.
7. Open `/log.csv` in your browser. Verify that the logged data points are appended correctly.

## Expected Output
Serial Monitor:
```
[SD] Mounted successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[SD] Initialized new CSV file.
[SD LOG] Saved data -> Time: 10, Temp: 24.0 C, Humid: 40.0%
[SD LOG] Saved data -> Time: 20, Temp: 28.5 C, Humid: 39.5%
```

## Expected Canvas Behavior
* Adjusting the simulated DHT22 climate values appends data packets to the SD card file.
* The web page reads the CSV and updates the graph trend line dynamically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `dht.readTemperature()` | Samples the temperature in Celsius from the DHT22. |
| `SD.open(..., FILE_APPEND)` | Opens the file in append mode to add data to the end. |
| `file.printf("%lu,%.1f,%.1f\n", ...)` | Formats the data point as a CSV row. |
| `fetch('/log.csv')` | Web browser requests the raw CSV log file over HTTP. |

## Hardware & Safety Concept: File Corruption and Non-volatile Buffers
* **File Corruption**: SD cards write data in sectors (usually 512 bytes). If power is cut while the ESP32 is writing a sector, the file table can get corrupted, destroying all logged data. To prevent this, keep the file open only while writing, and call `file.close()` immediately.
* **Write Buffering**: To prevent file system fragmentation and reduce power consumption, you can buffer readings in RAM and write them to the SD card in blocks (e.g. once every 5 minutes) instead of writing after every sample.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a graph of the last 10 readings.
2. **Humid Graph**: Modify the webpage script to plot both temperature and humidity on the same graph using different colors.
3. **Log Reset Button**: Add a button on the webpage to delete `/temp_log.csv` and start a new log.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The graph line is a flat line | No data points logged yet | Wait at least 20 seconds for the ESP32 to append the first few readings |
| SD Card Mount Failed | Card format wrong | Ensure the SD card is formatted as FAT16 or FAT32 |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS file logger](../intermediate/62-esp32-spiffs-file-logger-oled-diagnostics.md)
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [138 - ESP32 SD Card Energy Logger Web status](138-esp32-sd-card-energy-logger-web-status.md) (Next project)
