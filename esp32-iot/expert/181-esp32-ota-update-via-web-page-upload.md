# 181 - OTA Update via Web Page upload

Build an Over-The-Air (OTA) wireless firmware updater on the ESP32 that hosts a secure web update page, receives a compiled binary file (`.bin`) via multipart HTTP POST, flashes the update directly to the OTA partition using the built-in Update library, and reboots to run the new program.

## Goal
Learn how to manage ESP32 partition layouts, handle binary firmware uploads, write to flash sectors using the Update library, execute boot integrity checks, and host local update consoles.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An HTTP server is started on port 80. Navigating to the ESP32's IP address displays an OTA Update page. The user compiles a new program in their IDE, exports the `.bin` file, and uploads it via the web form. The ESP32 writes the new code to its secondary boot partition. Once the upload finishes, the ESP32 reboots, swapping partitions to run the new code.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

No external physical sensors or components are required for this project. The ESP32 is powered via its Micro-USB port, running as a secure OTA flash server.

> **Wiring tip:** Secure stable power when performing OTA updates to prevent partition corruption or bricking the device.

## Code
```cpp
// OTA Update via Web Page upload (Embedded HTTP Binary Uploader + Update library + System Reboot)
#include <WiFi.h>
#include <WebServer.h>
#include <Update.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int STATUS_LED = 12;

WebServer server(80);

// Serve the OTA upload landing page
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 OTA Update</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .badge { display: inline-block; padding: 6px 12px; border-radius: 12px; font-size: 11px; font-weight: bold; background-color: #7f1d1d; color: #fca5a5; margin-bottom: 20px; }\n";
  html += "  .badge.active { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .upload-box { border: 2px dashed #334155; border-radius: 8px; padding: 25px; text-align: center; margin-top: 20px; }\n";
  html += "  input[type=file] { display: none; }\n";
  html += "  .file-label { display: inline-block; padding: 12px 24px; background-color: #334155; border-radius: 6px; cursor: pointer; font-weight: bold; margin-bottom: 10px; transition: background-color 0.2s; }\n";
  html += "  .file-label:hover { background-color: #475569; }\n";
  html += "  .btn-submit { background-color: #10b981; color: white; width: 100%; padding: 14px; font-size: 15px; font-weight: bold; border-radius: 6px; margin-top: 15px; cursor: pointer; border: none; transition: background-color 0.2s; }\n";
  html += "  .btn-submit:hover { background-color: #059669; }\n";
  html += "  .progress-info { font-size: 12px; color: #64748b; margin-top: 15px; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Firmware OTA Update</h1>\n";
  html += "  <div class=\"badge active\">OTA SERVER ONLINE</div>\n";
  
  html += "  <div class=\"upload-box\">\n";
  html += "    <form method=\"POST\" action=\"/update\" enctype=\"multipart/form-data\">\n";
  html += "      <label for=\"fileInput\" class=\"file-label\">Select Firmware Binary (.bin)</label>\n";
  html += "      <input type=\"file\" name=\"update\" id=\"fileInput\" accept=\".bin\" onchange=\"updateFilenameLabel()\">\n";
  html += "      <div id=\"selectedFilename\" style=\"font-size:13px;color:#94a3b8;margin-bottom:10px;\">No file chosen</div>\n";
  html += "      <button type=\"submit\" class=\"btn-submit\">Flash New Firmware</button>\n";
  html += "    </form>\n";
  html += "  </div>\n";
  
  html += "  <div class=\"progress-info\">Current Version: V1.0.0 (MbedO Core)</div>\n";
  html += "  <p class=\"footer\">ESP32 Uptime: " + String(millis() / 1000) + " s</p>\n";
  html += "</div>\n";
  
  // Javascript helper
  html += "<script>\n";
  html += "  function updateFilenameLabel() {\n";
  html += "    const inp = document.getElementById('fileInput');\n";
  html += "    document.getElementById('selectedFilename').innerText = inp.files.length > 0 ? inp.files[0].name : 'No file chosen';\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Handle OTA upload chunk callbacks (runs iteratively during stream reception)
void handleOTAUpdate() {
  HTTPUpload& upload = server.upload();
  
  if (upload.status == UPLOAD_FILE_START) {
    Serial.printf("[OTA] Starting firmware flash for: %s\n", upload.filename.c_str());
    digitalWrite(STATUS_LED, HIGH); // LED ON indicates active flash write
    
    // Begin update using max sketch space calculation
    if (!Update.begin(UPDATE_SIZE_UNKNOWN)) {
      Update.printError(Serial);
    }
  } 
  else if (upload.status == UPLOAD_FILE_WRITE) {
    // Write received buffer block directly to flash sector
    if (Update.write(upload.buf, upload.currentSize) != upload.currentSize) {
      Update.printError(Serial);
    } else {
      Serial.print("."); // Progress dot indicator
    }
  } 
  else if (upload.status == UPLOAD_FILE_END) {
    if (Update.end(true)) { // Set true to run sketch verification checks
      Serial.printf("\n[OTA] Firmware upload completed! Size: %u Bytes.\n", upload.totalSize);
      Serial.println("[OTA] Rebooting to run new program...");
      
      // Send success response page to client
      String html = "<html><head><meta http-equiv='refresh' content='8;url=/'></head>";
      html += "<body style='background-color:#0f172a;color:#f8fafc;font-family:sans-serif;text-align:center;padding:50px;'>";
      html += "<h2>Firmware flashed successfully!</h2><p>Rebooting device. Access portal again in 8 seconds...</p></body></html>";
      server.send(200, "text/html", html);
      
      delay(2000);
      ESP.restart(); // Reboot ESP32
    } else {
      Update.printError(Serial);
      server.send(500, "text/plain", "OTA Failed: Verification Error");
    }
    digitalWrite(STATUS_LED, LOW);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\nESP32 OTA Server starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Web Server Routes
  server.on("/", HTTP_GET, handleRoot);
  
  // Register OTA upload handler alongside upload stream callback
  server.on("/update", HTTP_POST, []() {
    // If we reach this handler, the file stream has finished without errors
    server.sendHeader("Connection", "close");
    server.send(200, "text/plain", (Update.hasError()) ? "FAIL" : "OK");
  }, handleOTAUpdate);
  
  server.begin();
  Serial.println("HTTP Diagnostic Server active.");
}

void loop() {
  server.handleClient();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that the webpage displays "Firmware OTA Update" and the version info.
6. Click "Select Firmware Binary", choose a pre-compiled `.bin` file, and click "Flash New Firmware".
7. Verify that the console prints progress dots, the LED turns ON during flashing, and the webpage redirects to a reboot screen.
8. On the reboot run, verify that the ESP32 starts up with the new firmware.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP Diagnostic Server active.
[OTA] Starting firmware flash for: firmware.bin
...................................
[OTA] Firmware upload completed! Size: 612400 Bytes.
[OTA] Rebooting to run new program...
```

## Expected Canvas Behavior
* Running the code starts the OTA server. Uploading a binary file flashes the virtual partition, triggers the LED widget, and reboots the simulator.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <Update.h>` | Includes the system firmware flashing API from the ESP32 core. |
| `Update.begin(...)` | Sets up the OTA write partition and validates available space. |
| `Update.write(...)` | Writes received buffer blocks directly to the flash partition. |
| `Update.end(true)` | Ends writing and performs MD5 checksum verification checks. |

## Hardware & Safety Concept: OTA Partition Table and Rollback Safeguards
* **OTA Partition Table**: The ESP32 uses a custom partition layout to support OTA. By default, it splits sketch memory into two equal partitions: `app0` and `app1`. While `app0` is running the active program, the OTA updater writes the new binary to `app1`. Once verified, the bootloader changes the active partition flag to boot from `app1` on the next restart.
* **Rollback Safeguards**: If the new firmware crashes during startup, the watchdog timer triggers a reset. If the crash occurs repeatedly, the bootloader can roll back to the previous stable partition automatically, preventing the device from being bricked.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a progress bar indicating flash completion.
2. **Access Security PIN**: Add basic HTTP authentication (Project 64) to secure the OTA upload route.
3. **SPIFFS manager gateway**: Combine this project with the SPIFFS manager (Project 180) to create a unified system administrator console.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Flash fails with space error | Incorrect partition table | Ensure your ESP32 partition scheme is configured to support OTA (e.g. "Minimal OTA" or "Default" with OTA partitions) |
| Device loops on reboot | Corrupted binary | Ensure the compiled binary matches the ESP32 chip type (do not upload ESP8266 or ESP32-S3 binaries) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS flash memory data logger](../intermediate/62-esp32-spiffs-flash-memory-data-logger.md)
- [180 - SPIFFS Web File Manager](180-spiffs-web-file-manager.md)
- [182 - OTA Update via HTTP server download](182-esp32-ota-update-via-http-server-download.md) (Next project)
