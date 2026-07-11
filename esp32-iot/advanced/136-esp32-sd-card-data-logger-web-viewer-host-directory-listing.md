# 136 - SD Card Data Logger Web Viewer (Host directory listing)

Build a web-accessible SD card file server on the ESP32 that interfaces with an SD card module over SPI (CS on GPIO 5, SCK on GPIO 18, MISO on GPIO 19, MOSI on GPIO 23) using the `SD` library, and hosts a web page displaying a directory listing of files with download links.

## Goal
Learn how to initialize SD card modules over SPI, traverse directories using the SD library, list file names and sizes, stream files dynamically over HTTP, and serve web indexes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An SD card reader is connected via SPI. On boot, the ESP32 verifies the SD card mount and creates dummy log files if they are missing. Navigating to the ESP32's IP address displays a webpage showing a list of all files stored on the SD card. Clicking a file name downloads it directly to your computer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Micro SD Card Reader SPI Module | `sd_card` | Yes | Yes |
| Micro SD Card (FAT16/FAT32 formatted) | `sd_card` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, files are stored on a virtual SD card image, allowing complete interaction with the directory listing and downloading features.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| SD Card Module | CS (Chip Select) | GPIO5 | Yellow | SPI chip select |
| SD Card Module | SCK (Clock) | GPIO18 | Green | SPI clock output |
| SD Card Module | MISO (Data Out)| GPIO19 | Blue | SPI master input |
| SD Card Module | MOSI (Data In) | GPIO23 | White | SPI master output |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the SD Card module from the 5V Vin rail.

## Code
```cpp
// SD Card Data Logger Web Viewer (SPI SD Mount + Directory list + File Streamer)
#include <WiFi.h>
#include <WebServer.h>
#include <SPI.h>
#include <SD.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SD_CS_PIN = 5;

WebServer server(80);

// Write placeholder files to the SD card if empty
void createDummyLogs() {
  if (!SD.exists("/daily_log.csv")) {
    File f = SD.open("/daily_log.csv", FILE_WRITE);
    if (f) {
      f.println("Timestamp,TempC,Humidity");
      f.println("1719878400,24.5,55");
      f.println("1719882000,25.1,53");
      f.println("1719885600,25.8,50");
      f.close();
      Serial.println("[SD] Created daily_log.csv placeholder.");
    }
  }
  
  if (!SD.exists("/system.txt")) {
    File f = SD.open("/system.txt", FILE_WRITE);
    if (f) {
      f.println("System Log initialized.");
      f.println("WiFi: Connected");
      f.println("Status: Active");
      f.close();
      Serial.println("[SD] Created system.txt placeholder.");
    }
  }
}

// HTTP Handler to download files
void handleFileDownload() {
  if (server.hasArg("path")) {
    String filepath = server.arg("path");
    
    // Safety check: ensure file path starts with /
    if (!filepath.startsWith("/")) {
      filepath = "/" + filepath;
    }
    
    if (SD.exists(filepath)) {
      File f = SD.open(filepath, FILE_READ);
      
      // Determine content type
      String contentType = "application/octet-stream";
      if (filepath.endsWith(".txt")) contentType = "text/plain";
      else if (filepath.endsWith(".csv")) contentType = "text/csv";
      
      // Stream the file directly to the client
      server.streamFile(f, contentType);
      f.close();
      Serial.printf("[Server] Streamed file: %s\n", filepath.c_str());
    } else {
      server.send(404, "text/plain", "File Not Found");
    }
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage containing file directory list
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>SD Card File Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; text-align: center; }\n";
  html += "  .file-list { margin-top: 20px; border: 1px solid #334155; border-radius: 8px; overflow: hidden; background-color: #0f172a; }\n";
  html += "  .file-item { display: flex; justify-content: space-between; align-items: center; padding: 12px 16px; border-bottom: 1px solid #1e293b; font-size: 14px; }\n";
  html += "  .file-item:last-child { border-bottom: none; }\n";
  html += "  .file-name { color: #f1f5f9; text-decoration: none; font-weight: bold; }\n";
  html += "  .file-name:hover { color: #38bdf8; text-decoration: underline; }\n";
  html += "  .file-size { color: #64748b; font-family: monospace; }\n";
  html += "  .badge { display: block; text-align: center; padding: 8px; border-radius: 6px; font-weight: bold; background-color: #065f46; color: #a7f3d0; margin-bottom: 20px; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>SD Card File Console</h1>\n";
  html += "  <div class=\"badge\">SD CARD MOUNTED</div>\n";
  html += "  <h3>Stored Logs</h3>\n";
  html += "  <div class=\"file-list\">\n";
  
  // Read root directory and list files
  File root = SD.open("/");
  if (root) {
    File file = root.openNextFile();
    int count = 0;
    while (file) {
      if (!file.isDirectory()) {
        String name = String(file.name());
        
        // Strip leading slash if present
        if (name.startsWith("/")) {
          name = name.substring(1);
        }
        
        html += "    <div class=\"file-item\">\n";
        html += "      <a class=\"file-name\" href=\"/download?path=" + name + "\">" + name + "</a>\n";
        html += "      <span class=\"file-size\">" + String(file.size()) + " B</span>\n";
        html += "    </div>\n";
        count++;
      }
      file = root.openNextFile();
    }
    if (count == 0) {
      html += "    <div class=\"file-item\" style=\"color:#64748b; justify-content:center;\">No files found on SD card.</div>\n";
    }
    root.close();
  } else {
    html += "    <div class=\"file-item\" style=\"color:#ef4444; justify-content:center;\">Failed to read root directory.</div>\n";
  }
  
  html += "  </div>\n";
  html += "  <p class=\"footer\">Click on a file name to download | Port 80</p>\n";
  html += "</div>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nInitializing SD Card...");
  
  // 1. Initialize SD Card over SPI
  if (!SD.begin(SD_CS_PIN)) {
    Serial.println("[SD Error] Card Mount Failed! System halted.");
    for (;;);
  }
  Serial.println("[SD] Mounted successfully.");
  
  // 2. Write dummy files to test download functionality
  createDummyLogs();
  
  Serial.println("\nESP32 SD File Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/download", HTTP_GET, handleFileDownload);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **SD Card Module** onto the canvas.
2. Wire SD pins: CS to **GPIO5**, SCK to **GPIO18**, MISO to **GPIO19**, and MOSI to **GPIO23**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that `daily_log.csv` and `system.txt` display in the file list on the webpage.
6. Click `daily_log.csv`. Verify that the file downloads to your computer and contains the placeholder data.

## Expected Output
Serial Monitor:
```
Initializing SD Card...
[SD] Mounted successfully.
[SD] Created daily_log.csv placeholder.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Server] Streamed file: /daily_log.csv
```

## Expected Canvas Behavior
* The virtual SD card widget processes SPI read/write command blocks, populating the web index list.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SD.begin(SD_CS_PIN)` | Configures the SPI interface and mounts the SD card. |
| `root.openNextFile()` | Traverses files on the SD card sequentially. |
| `file.size()` | Retrieves the file size in bytes. |
| `server.streamFile(...)` | Streams the file contents directly over HTTP to the client browser. |

## Hardware & Safety Concept: SPI Sharing and Power Requirements
* **SPI Bus Sharing**: The SPI bus (SCK, MISO, MOSI) can be shared between multiple SPI devices (such as SD card modules and RFID readers) as long as each device has its own unique Chip Select (CS) pin. Make sure to set inactive CS pins HIGH before starting communication with the active device.
* **Power Requirements**: SD cards draw high current surges (up to 150 mA) when writing. Powering the SD module from the ESP32's 3.3V pin can cause voltage drops and corrupt files. Power the SD module from a clean 5V Vin rail via an onboard LDO regulator.

## Try This! (Challenges)
1. **Directory creation**: Extend the code to display files inside subdirectories recursively.
2. **File Deletion**: Add a delete button next to each file to allow deleting files from the SD card.
3. **Upload File**: Implement an HTTP POST upload form to allow uploading files from your computer to the SD card.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card Mount Failed | Card format wrong | Ensure the SD card is formatted as FAT16 or FAT32, not exFAT or NTFS |
| Files are corrupted | Loose connections | SPI signals are high frequency. Keep jumper wires under 10 cm |
| Webpage loads but directory is empty | No files found | Run the dummy initialization function to create the log files |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS file logger](../intermediate/62-esp32-spiffs-file-logger-oled-diagnostics.md)
- [135 - ESP32 Robotic Arm Web Position Memory Log](135-esp32-robotic-arm-web-position-memory-log.md)
- [137 - ESP32 SD Card Temp Logger Web Graph](137-esp32-sd-card-temp-logger-web-graph.md) (Next project)
