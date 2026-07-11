# 62 - Web Server serving files from SPIFFS (HTML files)

Build a web server on the ESP32 that mounts the internal SPI Flash File System (SPIFFS), reads static HTML assets stored in flash, and streams the files to client browsers using stream file APIs.

## Goal
Learn how to mount SPIFFS, open and read files, write files programmatically, and stream files to clients with correct MIME types.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It mounts the SPIFFS filesystem and checks for `/index.html`. If missing (e.g. on first boot), the ESP32 programmatically creates the file and writes a template HTML page. Navigating to the ESP32's IP address opens `/index.html`, streams its content to the browser, and displays the page.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Web Server serving files from SPIFFS (Mounting & streaming static files)
#include <WiFi.h>
#include <WebServer.h>
#include <FS.h>
#include <SPIFFS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// Helper function to create a test HTML file in SPIFFS if it doesn't exist
void prepareFileSystemFiles() {
  if (!SPIFFS.exists("/index.html")) {
    Serial.println("[SPIFFS] /index.html not found. Creating file...");
    
    // 1. Open file in Write Mode ("w")
    File file = SPIFFS.open("/index.html", "w");
    
    if (file) {
      // Write HTML content
      file.println("<!DOCTYPE html>");
      file.println("<html>");
      file.println("<head>");
      file.println("  <meta name='viewport' content='width=device-width, initial-scale=1.0'>");
      file.println("  <title>SPIFFS File Server</title>");
      file.println("  <style>");
      file.println("    body { font-family: sans-serif; background-color: #0f172a; color: #f8fafc; text-align: center; padding-top: 50px; }");
      file.println("    .card { display: inline-block; background-color: #1e293b; border-radius: 12px; padding: 40px; border: 1px solid #334155; box-shadow: 0 10px 15px rgba(0,0,0,0.5); }");
      file.println("    h1 { color: #38bdf8; }");
      file.println("  </style>");
      file.println("</head>");
      file.println("<body>");
      file.println("  <div class='card'>");
      file.println("    <h1>SPIFFS File Server</h1>");
      file.println("    <p>This page is stored as a file in the ESP32's flash memory file system.</p>");
      file.println("    <p>The code reads this file from flash and streams it to your browser.</p>");
      file.println("  </div>");
      file.println("</body>");
      file.println("</html>");
      
      file.close();
      Serial.println("[SPIFFS] /index.html created successfully.");
    } else {
      Serial.println("[SPIFFS Error] Failed to open /index.html for writing!");
    }
  } else {
    Serial.println("[SPIFFS] /index.html already exists. Skipping creation.");
  }
}

// Root URL Route Handler ("/")
void handleRoot() {
  Serial.println("[Server] Request received for /");
  
  // 2. Open the index file in Read Mode ("r")
  if (SPIFFS.exists("/index.html")) {
    File file = SPIFFS.open("/index.html", "r");
    
    // 3. Stream the file directly to the client browser
    // This is faster and uses less memory than reading the file into a String
    server.streamFile(file, "text/html");
    
    file.close();
    Serial.println("[Server] /index.html streamed successfully.");
  } else {
    server.send(404, "text/plain", "Error: File /index.html not found on SPIFFS!");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 SPIFFS Web Server Starting");
  Serial.println("==================================");
  
  // 4. Mount SPIFFS
  // The parameter 'true' tells SPIFFS to format the filesystem if mounting fails
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Failed to mount file system!");
    while (1) {}
  }
  Serial.println("[SPIFFS] File system mounted successfully.");
  
  // Prepare files
  prepareFileSystemFiles();
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  Serial.println("==================================\n");
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open your web browser and navigate to the printed IP address. Verify that the webpage stored in SPIFFS loads correctly.

## Expected Output
Serial Monitor:
```
==================================
ESP32 SPIFFS Web Server Starting
==================================
[SPIFFS] File system mounted successfully.
[SPIFFS] /index.html not found. Creating file...
[SPIFFS] /index.html created successfully.
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
==================================

[Server] Request received for /
[Server] /index.html streamed successfully.
```

## Expected Canvas Behavior
* Navigating to the page opens and streams `/index.html` from the flash filesystem, logging the operations on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SPIFFS.begin(true)` | Mounts the flash file system, formatting it automatically if it is unformatted. |
| `SPIFFS.open(..., "w")` | Creates and opens a file in write mode to write template HTML text. |
| `SPIFFS.open(..., "r")` | Opens an existing file in read-only mode. |
| `server.streamFile(...)` | Streams the open file contents directly to the network socket buffer. |

## Hardware & Safety Concept: Flash File Systems and RAM Allocations
* **In-code Strings**: Storing large HTML pages as strings inside C++ variables occupies RAM. Compiling pages using string concatenation (e.g. `html += "..."`) can cause heap fragmentation and crash the microcontroller.
* **SPIFFS**: Storing pages in flash memory (SPIFFS) allows reading and streaming files directly to the network buffer via `server.streamFile()`, preserving RAM.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the free space remaining in SPIFFS.
2. **File Lister API**: Add an API route `/list` that reads the SPIFFS root directory and returns a JSON list of all stored files.
3. **SPIFFS Logger**: Write code to append a timestamp to a log file `/log.txt` in SPIFFS every time a client visits the webpage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Mount fails permanently | Flash layout incorrect | Select a partition layout that allocates space for SPIFFS (e.g. Default 4MB with SPIFFS) |
| Files disappear after reset | Mount parameter incorrect | Ensure `SPIFFS.begin(true)` is called so the system mounts the partition on boot |
| The webpage is slow to load | File streamed in small chunks | Use `server.streamFile()` to stream files directly to the socket |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [58 - ESP32 Web Server serving CSS styles](58-esp32-web-server-serving-css-styles.md)
- [63 - ESP32 Web Server serving images from SPIFFS](63-esp32-web-server-serving-images-from-spiffs.md) (Next project)
