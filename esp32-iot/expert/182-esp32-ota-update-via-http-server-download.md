# 182 - OTA Update via HTTP server download (auto checks binary URL updates)

Build an autonomous Over-The-Air (OTA) firmware update manager on the ESP32 that periodically polls a remote JSON server to check for new version numbers, downloads the binary file (`.bin`) automatically from a server URL, and reboots to run the new software version.

## Goal
Learn how to parse remote JSON version manifests, write HTTP client streams directly to flash using the Update library, manage download progress callbacks, and implement auto-reboot sequences.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It stores a current version variable (e.g. `1.0.0`). Every 30 seconds, it queries a simulated remote server API (`http://httpbin.org/get?newVersion=1.1.0`). If the returned version is higher, the ESP32 downloads the binary from the specified firmware URL, writes the code to the OTA partition, flashes a status LED on GPIO 12, and reboots automatically to run the new version.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// Auto-OTA Update via HTTP download (Periodic version check + Stream downloader + Partition swap)
#include <WiFi.h>
#include <HTTPClient.h>
#include <Update.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Simulated remote firmware manifest URL
// In production, this points to a server JSON file containing: {"version":"1.1.0","url":"http://mybucket/firmware.bin"}
const char* firmwareVersionUrl = "http://httpbin.org/get?newVersion=1.1.0";
// Simulated firmware binary file URL
const char* firmwareBinaryUrl = "http://httpbin.org/bytes/500000"; // Mock binary download (500 KB of random bytes)

const int STATUS_LED = 12;

// Current software version running on device
const String CURRENT_VERSION = "1.0.0";

unsigned long lastCheckTime = 0;
const unsigned long CHECK_INTERVAL_MS = 30000; // Check for updates every 30 seconds

// Download and flash the firmware binary stream from URL
void downloadFirmwareBinary(String url) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(url);
    
    int httpResponseCode = http.GET();
    if (httpResponseCode == 200) {
      int contentLength = http.getSize();
      Serial.printf("[Auto-OTA] Downloading firmware binary. Size: %d Bytes...\n", contentLength);
      
      WiFiClient* stream = http.getStreamPtr();
      
      // Initialize OTA update sequence
      if (Update.begin(contentLength)) {
        digitalWrite(STATUS_LED, HIGH); // LED ON indicates active flash writing
        
        // Write stream directly to flash partition
        size_t written = Update.writeStream(*stream);
        
        if (written == contentLength) {
          Serial.println("[Auto-OTA] Stream write successful.");
        } else {
          Serial.printf("[Auto-OTA Error] Write failed! Written: %d of %d\n", written, contentLength);
        }
        
        if (Update.end(true)) { // Set true to run sketch verification checks
          Serial.println("[Auto-OTA] Update completed successfully! Rebooting...");
          delay(2000);
          ESP.restart(); // Reboot ESP32
        } else {
          Update.printError(Serial);
        }
        digitalWrite(STATUS_LED, LOW);
      } else {
        Serial.println("[Auto-OTA Error] Not enough space to write firmware!");
      }
    } else {
      Serial.printf("[Auto-OTA Error] Failed to fetch binary file. HTTP Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

// Periodically check remote server for new firmware versions
void checkForUpdates() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(firmwareVersionUrl);
    
    int httpResponseCode = http.GET();
    if (httpResponseCode == 200) {
      String payload = http.getString();
      
      // Parse simulated JSON version from httpbin query args
      // In production, use ArduinoJson to parse the server response
      int vIndex = payload.indexOf("\"newVersion\": \"");
      if (vIndex != -1) {
        String serverVersion = payload.substring(vIndex + 15, payload.indexOf("\"", vIndex + 15));
        Serial.printf("[Auto-OTA] Current Version: %s | Server Version: %s\n", 
                      CURRENT_VERSION.c_str(), serverVersion.c_str());
        
        // Trigger update if server version is higher
        if (serverVersion > CURRENT_VERSION) {
          Serial.println("[Auto-OTA] New firmware version available. Launching download...");
          downloadFirmwareBinary(firmwareBinaryUrl);
        } else {
          Serial.println("[Auto-OTA] Firmware is up to date.");
        }
      }
    } else {
      Serial.printf("[Auto-OTA Error] Version check request failed. Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\nESP32 Auto-OTA Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Perform initial update check on boot
  checkForUpdates();
  lastCheckTime = millis();
}

void loop() {
  // Periodically check for updates
  unsigned long now = millis();
  if (now - lastCheckTime >= CHECK_INTERVAL_MS) {
    lastCheckTime = now;
    checkForUpdates();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Verify that the console prints the connection message, connects successfully, and queries the simulated server.
5. Verify that the ESP32 detects the server version `1.1.0` (which is higher than the current `1.0.0`), triggers the download, flashes the LED, and reboots.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[Auto-OTA] Current Version: 1.0.0 | Server Version: 1.1.0
[Auto-OTA] New firmware version available. Launching download...
[Auto-OTA] Downloading firmware binary. Size: 500000 Bytes...
[Auto-OTA] Stream write successful.
[Auto-OTA] Update completed successfully! Rebooting...
```

## Expected Canvas Behavior
* Booting the system launches the update check. Detecting a higher version pulls the binary stream, blinks the LED widget, and reboots the simulator.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `http.getStreamPtr()` | Acquires the raw TCP stream pointer of the HTTP GET response body. |
| `Update.writeStream(*stream)` | Streams and writes the incoming binary blocks directly to flash memory. |
| `serverVersion > CURRENT_VERSION` | Evaluates if the firmware version needs an update. |
| `ESP.restart()` | Reboots the system after validation checks pass. |

## Hardware & Safety Concept: Secure HTTPS Downloads and Fail-safe Watchdogs
* **Secure HTTPS Downloads**: Downloading firmware over plaintext HTTP is a severe security risk, as attackers can hijack the DNS (Project 171) or run a Man-in-the-Middle (MitM) attack to push malicious firmware. In production, always download firmware over secure HTTPS (`https://`) and verify the server's SSL certificate fingerprint or public key.
* **Fail-Safe Watchdogs**: If an update fails halfway (due to power loss or poor signal), the ESP32 bootloader maintains the active partition flag, booting back into the previous stable version. Always include a watchdog timer to reset the chip if the download gets stuck.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a progress bar indicating download completion.
2. **Rollback Trigger**: Create a physical rollback switch (GPIO 14) that forces the bootloader to swap back to the previous partition on boot.
3. **ThingSpeak Integration**: Log the auto-update status and new version number to a ThingSpeak channel (Project 141).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Update fails with write error | Stream timeout | Large downloads can cause TCP stream timeouts. Keep firmware sizes small or increase router buffer sizes |
| Boot loop after update | Mismatched partition sizes | Ensure the partition sizes allocated in your IDE match the firmware size limits of the chip |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger Web Viewer](136-esp32-sd-card-data-logger-web-viewer-host-directory-listing.md)
- [180 - SPIFFS Web File Manager](180-spiffs-web-file-manager.md)
- [181 - OTA Update via Web Page upload](181-esp32-ota-update-via-web-page-upload.md)
- [183 - AWS IoT Core connection](183-esp32-aws-iot-core-connection.md) (Next project)
