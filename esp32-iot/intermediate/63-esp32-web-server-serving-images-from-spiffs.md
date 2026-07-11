# 63 - Web Server serving images from SPIFFS

Build a web server on the ESP32 that mounts the internal SPI Flash File System (SPIFFS), programmatically generates an SVG vector graphic asset, and serves it with the correct MIME type (`image/svg+xml`) to be rendered dynamically on a control page.

## Goal
Learn how to store and serve graphic assets, configure Content-Type headers for images (`image/svg+xml`), construct SVG code, and stream files.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and mounts the SPIFFS filesystem. It checks for a vector image file `/logo.svg`. If missing, the ESP32 programmatically writes the SVG XML string to the flash filesystem. Navigating to the ESP32's IP address loads a webpage that displays this vector logo.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// Web Server serving images from SPIFFS (SVG Vector Graphic Serving)
#include <WiFi.h>
#include <WebServer.h>
#include <FS.h>
#include <SPIFFS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// Helper function to create vector graphic and HTML index in SPIFFS if missing
void prepareStaticAssets() {
  // 1. Create Vector Graphic file if missing
  if (!SPIFFS.exists("/logo.svg")) {
    Serial.println("[SPIFFS] /logo.svg not found. Creating file...");
    File file = SPIFFS.open("/logo.svg", "w");
    if (file) {
      // Write XML SVG vector representation of an ESP32 microchip node logo
      file.println("<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 100 100' width='100' height='100'>");
      file.println("  <rect x='20' y='20' width='60' height='60' rx='10' fill='#1e293b' stroke='#38bdf8' stroke-width='4'/>");
      file.println("  <circle cx='50' cy='50' r='15' fill='#38bdf8'/>");
      file.println("  <rect x='10' y='30' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("  <rect x='10' y='46' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("  <rect x='10' y='62' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("  <rect x='80' y='30' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("  <rect x='80' y='46' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("  <rect x='80' y='62' width='10' height='8' rx='2' fill='#64748b'/>");
      file.println("</svg>");
      file.close();
      Serial.println("[SPIFFS] /logo.svg created successfully.");
    }
  }

  // 2. Create HTML Index file if missing
  if (!SPIFFS.exists("/index.html")) {
    Serial.println("[SPIFFS] /index.html not found. Creating file...");
    File file = SPIFFS.open("/index.html", "w");
    if (file) {
      file.println("<!DOCTYPE html>");
      file.println("<html>");
      file.println("<head>");
      file.println("  <meta name='viewport' content='width=device-width, initial-scale=1.0'>");
      file.println("  <title>SPIFFS Image Server</title>");
      file.println("  <style>");
      file.println("    body { font-family: sans-serif; background-color: #0f172a; color: #f8fafc; text-align: center; padding-top: 50px; }");
      file.println("    .card { display: inline-block; background-color: #1e293b; border-radius: 12px; padding: 40px; border: 1px solid #334155; box-shadow: 0 10px 15px rgba(0,0,0,0.5); }");
      file.println("    h1 { color: #38bdf8; margin-top: 20px; }");
      file.println("    img { display: block; margin: 0 auto 20px auto; }");
      file.println("  </style>");
      file.println("</head>");
      file.println("<body>");
      file.println("  <div class='card'>");
      // Reference to the SVG image route
      file.println("    <img src='/logo.svg' alt='ESP32 Node Logo' width='100' height='100'>");
      file.println("    <h1>SPIFFS Image Server</h1>");
      file.println("    <p>This web page and the SVG logo displayed above are stored as files in SPIFFS.</p>");
      file.println("  </div>");
      file.println("</body>");
      file.println("</html>");
      file.close();
      Serial.println("[SPIFFS] /index.html created successfully.");
    }
  }
}

// Root URL Route Handler ("/")
void handleRoot() {
  if (SPIFFS.exists("/index.html")) {
    File file = SPIFFS.open("/index.html", "r");
    server.streamFile(file, "text/html");
    file.close();
  } else {
    server.send(404, "text/plain", "Error: /index.html not found!");
  }
}

// SVG Logo URL Handler ("/logo.svg")
void handleLogo() {
  if (SPIFFS.exists("/logo.svg")) {
    // 3. Open image file from SPIFFS
    File file = SPIFFS.open("/logo.svg", "r");
    
    // 4. Configure caching headers to reduce network requests
    server.sendHeader("Cache-Control", "max-age=86400");
    
    // 5. Stream vector graphic with correct image/svg+xml MIME type
    server.streamFile(file, "image/svg+xml");
    
    file.close();
    Serial.println("[Server] /logo.svg served successfully.");
  } else {
    server.send(404, "text/plain", "Error: /logo.svg not found!");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 SPIFFS Asset Server Starting");
  Serial.println("==================================");
  
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Failed to mount file system!");
    while (1) {}
  }
  Serial.println("[SPIFFS] File system mounted.");
  
  // Create test assets
  prepareStaticAssets();
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/logo.svg", handleLogo);
  
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
3. Open your web browser and navigate to the printed IP address. Verify that the webpage and the vector logo load correctly.
4. Inspect the image tag to confirm that `/logo.svg` is loaded from the ESP32.

## Expected Output
Serial Monitor:
```
==================================
ESP32 SPIFFS Asset Server Starting
==================================
[SPIFFS] File system mounted.
[SPIFFS] /logo.svg not found. Creating file...
[SPIFFS] /logo.svg created successfully.
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
==================================

[Server] /logo.svg served successfully.
```

Browser Page Layout:
* A dark-themed card container displaying an SVG vector graphic of an ESP32 chip with connection lines above the heading.

## Expected Canvas Behavior
* Navigating to the page opens and streams `/logo.svg` from the flash filesystem, logging the operations on the Serial Monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `<img src='/logo.svg' ...>` | Instructs the browser to send an independent HTTP GET request to `/logo.svg` to fetch the image. |
| `server.streamFile(file, "image/svg+xml")` | Serves the SVG vector file with the correct XML graphic MIME header. |
| `Cache-Control` | Adds caching headers to instruct the browser to store the image locally, reducing queries. |

## Hardware & Safety Concept: Vector Graphics (SVG) vs Bitmaps (BMP/PNG) in IoT
Microcontrollers have limited flash space. Storing raw bitmap images (such as BMP or PNG files) requires significant space (a simple 100x100 PNG can occupy several kilobytes). **SVG (Scalable Vector Graphics)** files represent images as XML code structures (math lines and shapes). SVGs are text-based, highly scalable, and occupy a fraction of the space of bitmaps, making them ideal for IoT servers.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the count of image requests.
2. **Dynamic Logo Color**: Read a potentiometer (Project 34) and adjust the circle color inside the SVG string dynamically.
3. **Multi-image slider**: Store multiple SVG files in SPIFFS and cycle between them on the page.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage loads but the image is a broken icon | Content-Type header wrong | Verify that the `/logo.svg` response uses the `image/svg+xml` MIME type |
| Logo does not change after editing | Browser caching | Clear your browser cache or open the page in an incognito window |
| File write fails | SPIFFS filesystem full | Check free space using `SPIFFS.totalBytes() - SPIFFS.usedBytes()` |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [62 - ESP32 Web Server serving files from SPIFFS](62-esp32-web-server-serving-files-from-spiffs.md)
- [64 - ESP32 Web Server HTTP basic authentication lock](64-esp32-web-server-http-basic-authentication-lock.md) (Next project)
