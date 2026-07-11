# 180 - SPIFFS Web File Manager (Upload/Download/Delete files in browser)

Build an interactive local web file system explorer on the ESP32 that mounts the SPIFFS flash partition, hosts an administrative HTTP portal, and enables users to browse, download, delete, and upload files directly in their browser.

## Goal
Learn how to iterate through directory entries in SPIFFS, handle HTML multipart file uploads, stream binary files over HTTP, calculate storage space usage, and build file dashboards.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It mounts the SPIFFS filesystem. An HTTP server is started on port 80. Navigating to the ESP32's IP address displays a file explorer page showing a list of all files stored in flash, their sizes, and a progress bar representing total vs. used storage. The page includes buttons to download or delete files, and an upload field to push new files directly to the ESP32.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

No external physical sensors or components are required for this project. The ESP32 is powered via its Micro-USB port, running as a local fileserver.

> **Wiring tip:** Secure stable power when performing file system writes to prevent partition corruption.

## Code
```cpp
// SPIFFS Web File Manager (Directory Lister + Multipart File Uploader + Stream Downloader + File Remover)
#include <WiFi.h>
#include <WebServer.h>
#include <SPIFFS.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// Global file stream object for active uploads
File uploadFile;

// Format file size units
String formatBytes(size_t bytes) {
  if (bytes < 1024) return String(bytes) + " B";
  else if (bytes < (1024 * 1024)) return String(bytes / 1024.0, 1) + " KB";
  else return String(bytes / (1024.0 * 1024.0), 1) + " MB";
}

// Serve root webpage containing file lists and upload form
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>SPIFFS File Explorer</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 520px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .storage-bar { height: 10px; background-color: #334155; border-radius: 5px; margin: 20px 0; overflow: hidden; }\n";
  html += "  .storage-fill { height: 100%; background-color: #10b981; }\n";
  html += "  .storage-info { font-size: 12px; color: #94a3b8; display: flex; justify-content: space-between; margin-bottom: 25px; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin: 20px 0; font-size: 14px; }\n";
  html += "  th { text-align: left; color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 10px 0; border-bottom: 1px solid #1e293b; color: #f1f5f9; }\n";
  html += "  .btn { padding: 6px 12px; font-size: 12px; font-weight: bold; border-radius: 4px; border: none; cursor: pointer; text-decoration: none; display: inline-block; }\n";
  html += "  .btn-download { background-color: #38bdf8; color: #0f172a; margin-right: 5px; }\n";
  html += "  .btn-delete { background-color: #ef4444; color: white; }\n";
  html += "  .upload-box { border: 2px dashed #334155; border-radius: 8px; padding: 20px; text-align: center; margin-top: 25px; }\n";
  html += "  input[type=file] { display: none; }\n";
  html += "  .file-label { display: inline-block; padding: 10px 20px; background-color: #334155; border-radius: 6px; cursor: pointer; font-weight: bold; margin-bottom: 10px; }\n";
  html += "  .btn-submit { background-color: #10b981; color: white; width: 100%; padding: 12px; font-size: 14px; font-weight: bold; border-radius: 6px; margin-top: 10px; cursor: pointer; border: none; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>SPIFFS File Explorer</h1>\n";
  
  // Storage usage calculator
  size_t totalBytes = SPIFFS.totalBytes();
  size_t usedBytes = SPIFFS.usedBytes();
  float usagePct = ((float)usedBytes / totalBytes) * 100.0;
  
  html += "  <div class=\"storage-bar\"><div class=\"storage-fill\" style=\"width:" + String(usagePct, 1) + "%\"></div></div>\n";
  html += "  <div class=\"storage-info\"><span>Used: " + formatBytes(usedBytes) + "</span><span>Total: " + formatBytes(totalBytes) + "</span></div>\n";
  
  html += "  <table>\n";
  html += "    <thead><tr><th>File Name</th><th>Size</th><th>Actions</th></tr></thead>\n";
  html += "    <tbody>\n";
  
  // List directory contents
  File root = SPIFFS.open("/");
  File file = root.openNextFile();
  int fileCount = 0;
  while (file) {
    fileCount++;
    String filename = String(file.name());
    // Strip leading slash if present
    if (filename.startsWith("/")) filename = filename.substring(1);
    
    html += "      <tr>\n";
    html += "        <td>" + filename + "</td>\n";
    html += "        <td>" + formatBytes(file.size()) + "</td>\n";
    html += "        <td>\n";
    html += "          <a href=\"/download?file=" + filename + "\" class=\"btn btn-download\">Get</a>\n";
    html += "          <a href=\"/delete?file=" + filename + "\" class=\"btn btn-delete\" onclick=\"return confirm('Delete this file?')\">Del</a>\n";
    html += "        </td>\n";
    html += "      </tr>\n";
    file = root.openNextFile();
  }
  
  if (fileCount == 0) {
    html += "      <tr><td colspan=\"3\" style=\"text-align:center;color:#64748b;padding:20px;\">No files stored in flash.</td></tr>\n";
  }
  
  html += "    </tbody>\n";
  html += "  </table>\n";
  
  // Upload form
  html += "  <div class=\"upload-box\">\n";
  html += "    <form method=\"POST\" action=\"/upload\" enctype=\"multipart/form-data\">\n";
  html += "      <label for=\"fileInput\" class=\"file-label\">Choose File</label>\n";
  html += "      <input type=\"file\" name=\"upload\" id=\"fileInput\" onchange=\"updateFilenameLabel()\">\n";
  html += "      <div id=\"selectedFilename\" style=\"font-size:12px;color:#64748b;margin-bottom:10px;\">No file chosen</div>\n";
  html += "      <button type=\"submit\" class=\"btn btn-submit\">Upload File to ESP32</button>\n";
  html += "    </form>\n";
  html += "  </div>\n";
  
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

// Stream download files from SPIFFS
void handleFileDownload() {
  if (server.hasArg("file")) {
    String filename = "/" + server.arg("file");
    if (SPIFFS.exists(filename)) {
      File file = SPIFFS.open(filename, FILE_READ);
      // Stream file with octet binary header configuration
      server.streamFile(file, "application/octet-stream");
      file.close();
      Serial.println("[SPIFFS] Streamed download file: " + filename);
    } else {
      server.send(404, "text/plain", "File Not Found");
    }
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Remove files from SPIFFS
void handleFileDelete() {
  if (server.hasArg("file")) {
    String filename = "/" + server.arg("file");
    if (SPIFFS.exists(filename)) {
      SPIFFS.remove(filename);
      Serial.println("[SPIFFS] Deleted file: " + filename);
      
      // Redirect back to root page
      server.sendHeader("Location", "/", true);
      server.send(303);
    } else {
      server.send(404, "text/plain", "File Not Found");
    }
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Handle file upload chunk callbacks (runs iteratively during stream reception)
void handleFileUpload() {
  HTTPUpload& upload = server.upload();
  
  if (upload.status == UPLOAD_FILE_START) {
    String filename = upload.filename;
    if (!filename.startsWith("/")) {
      filename = "/" + filename;
    }
    Serial.printf("[SPIFFS UPLOAD] Starting upload for: %s\n", filename.c_str());
    
    // Open file in SPIFFS to write incoming blocks
    uploadFile = SPIFFS.open(filename, FILE_WRITE);
  } 
  else if (upload.status == UPLOAD_FILE_WRITE) {
    if (uploadFile) {
      uploadFile.write(upload.buf, upload.currentSize);
    }
  } 
  else if (upload.status == UPLOAD_FILE_END) {
    if (uploadFile) {
      uploadFile.close();
      Serial.printf("[SPIFFS UPLOAD] Upload completed. Size: %d Bytes\n", upload.totalSize);
    }
    
    // Redirect back to root page after completion
    server.sendHeader("Location", "/", true);
    server.send(303);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Mount SPIFFS
  Serial.println("\nMounting SPIFFS filesystem...");
  if (!SPIFFS.begin(true)) {
    Serial.println("[SPIFFS Error] Mount failed! Formatting partition...");
  } else {
    Serial.println("[SPIFFS] Mounted successfully.");
  }
  
  Serial.println("\nESP32 File Manager starting...");
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
  server.on("/download", HTTP_GET, handleFileDownload);
  server.on("/delete", HTTP_GET, handleFileDelete);
  
  // Register file upload handler alongside upload stream callback
  server.on("/upload", HTTP_POST, []() {
    server.send(200, "text/plain", "Upload OK");
  }, handleFileUpload);
  
  server.begin();
  Serial.println("HTTP Server active.");
}

void loop() {
  server.handleClient();
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Open your web browser and navigate to the printed IP address.
4. Verify that the webpage displays the files list, showing "No files stored in flash." if empty.
5. Click "Choose File", select a small text file on your computer, and click "Upload File to ESP32".
6. Verify that the table refreshes, showing the newly uploaded file and its size.
7. Click "Get" next to the file to verify it downloads, or "Del" to delete it.

## Expected Output
Serial Monitor:
```
[SPIFFS] Mounted successfully.
WiFi Connected.
Local IP Address: 10.10.0.3
HTTP Server active.
[SPIFFS UPLOAD] Starting upload for: /test.txt
[SPIFFS UPLOAD] Upload completed. Size: 42 Bytes
[SPIFFS] Streamed download file: /test.txt
[SPIFFS] Deleted file: /test.txt
```

## Expected Canvas Behavior
* Running the code starts the file server. Uploading, downloading, and deleting files via the web dashboard updates the SPIFFS partition and terminal output.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `HTTPUpload& upload = server.upload()` | Accesses the active multipart file upload stream. |
| `SPIFFS.open(..., FILE_WRITE)` | Opens a file object in write mode to save incoming stream chunks. |
| `uploadFile.write(...)` | Writes received buffer blocks directly to flash memory. |
| `server.streamFile(...)` | Streams a file from SPIFFS back to the client browser. |

## Hardware & Safety Concept: Flash Block Size and Formatting Issues
* **Flash Block Size**: SPIFFS splits flash space into blocks and pages. When a file is modified, the whole sector must be erased and rewritten. If the storage partition is nearly full, these rewrite cycles can slow down significantly (write amplification). Try to keep at least 20% of the flash space empty to prevent performance degradation.
* **Formatting Issues**: If the partition table changes or a write fails due to power loss, SPIFFS can become corrupted. By calling `SPIFFS.begin(true)`, the ESP32 will automatically format the partition on boot if mounting fails, restoring functionality at the cost of erasing saved files.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the number of files stored in flash.
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound a warning chime if an upload fails or the storage space is full.
3. **Basic Authentication**: Protect the file manager page with an HTTP Basic Authentication lock (Project 64) to prevent unauthorized access.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Upload fails with empty file | Upload timeout | Large file uploads can take too long, exceeding the HTTP timeout. Keep uploaded files small (under 100 KB) |
| Downloaded file is corrupted | Binary vs. text mode | Ensure you are opening files using binary mode (`FILE_READ`) and streaming with the `application/octet-stream` header |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [62 - ESP32 SPIFFS flash memory data logger](../intermediate/62-esp32-spiffs-flash-memory-data-logger.md)
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [181 - OTA Update via Web Page upload](181-esp32-ota-update-via-web-page-upload.md) (Next project)
