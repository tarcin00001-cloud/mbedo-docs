# 45 - ESP32 Web-Based Buzzer Chime Trigger

Build a local web server on the ESP32 that hosts a control page, listens for URL parameter triggers (`/chime1`, `/chime2`, and `/siren`), and triggers distinct buzzer sound patterns on GPIO 4.

## Goal
Learn how to configure URL route handlers to actuate warning beeps, serve HTML pages, and build remote alarms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An active buzzer is on GPIO 4. Navigating to the ESP32's IP address in a web browser displays a page with buttons to trigger three sound patterns: a single beep (`/chime1`), a double chime (`/chime2`), and a rapid alert siren (`/siren`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Web-triggered alarm |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the active buzzer directly to GPIO 4.

## Code
```cpp
// Web-based Buzzer chime trigger (Web Server Sound Controller)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int BUZZER_PIN = 4;

WebServer server(80);

void playBuzzerPattern(int count, int durationMs, int gapMs) {
  for (int i = 0; i < count; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(durationMs);
    digitalWrite(BUZZER_PIN, LOW);
    if (i < count - 1) {
      delay(gapMs);
    }
  }
}

// 1. Root URL handler ("/")
void handleRoot() {
  String html = "<html><head><title>ESP32 Web Buzzer</title>";
  html += "<style>body { font-family: sans-serif; text-align: center; margin-top: 50px; }";
  html += "a { display: inline-block; padding: 15px 25px; margin: 10px; font-size: 18px; color: white; text-decoration: none; border-radius: 5px; }";
  html += ".c1 { background-color: #3a86c8; } .c2 { background-color: #ffb703; } .sir { background-color: #fb8500; }</style></head>";
  html += "<body><h1>ESP32 Web Buzzer Controller</h1>";
  html += "<p>Click a button below to trigger a sound pattern remotely:</p>";
  html += "<a href=\"/chime1\" class=\"c1\">Chime 1: Single Beep</a><br>";
  html += "<a href=\"/chime2\" class=\"c2\">Chime 2: Double Chirp</a><br>";
  html += "<a href=\"/siren\" class=\"sir\">Siren: Emergency Pulse</a>";
  html += "</body></html>";
  
  server.send(200, "text/html", html);
}

// 2. Chime 1 handler ("/chime1")
void handleChime1() {
  Serial.println("[Web Action] Triggered Chime 1 (Single Beep).");
  server.sendHeader("Location", "/");
  server.send(303); // Redirect back to control panel
  
  playBuzzerPattern(1, 200, 0);
}

// 3. Chime 2 handler ("/chime2")
void handleChime2() {
  Serial.println("[Web Action] Triggered Chime 2 (Double Chirp).");
  server.sendHeader("Location", "/");
  server.send(303);
  
  playBuzzerPattern(2, 80, 80);
}

// 4. Siren handler ("/siren")
void handleSiren() {
  Serial.println("[Web Action] Triggered Siren (Emergency Pulse).");
  server.sendHeader("Location", "/");
  server.send(303);
  
  playBuzzerPattern(5, 50, 50);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\n==================================");
  Serial.println("ESP32 Web Buzzer Control Setup");
  Serial.println("==================================");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Register URL routes
  server.on("/", handleRoot);
  server.on("/chime1", handleChime1);
  server.on("/chime2", handleChime2);
  server.on("/siren", handleSiren);
  
  // Start server
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
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Wire Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. Note the printed IP address (e.g. `http://10.10.0.3`).
5. Open your web browser and navigate to the IP. Click the buttons to trigger the buzzer chime patterns.

## Expected Output
Serial Monitor:
```
==================================
ESP32 Web Buzzer Control Setup
==================================
WiFi Connected successfully.
Web Server active. Connect at: http://10.10.0.3
==================================

[Web Action] Triggered Chime 1 (Single Beep).
[Web Action] Triggered Chime 2 (Double Chirp).
[Web Action] Triggered Siren (Emergency Pulse).
```

## Expected Canvas Behavior
* Clicking the buttons on the web page pulses the buzzer widget green.
* The Serial Monitor logs the client requests.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WebServer server(80)` | Instantiates a web server object listening on standard HTTP port 80. |
| `server.on("/chime1", ...)` | Binds URL route paths to their handler functions. |
| `playBuzzerPattern(...)` | Drives the buzzer pin to generate the warning chimes. |
| `server.send(303)` | Redirects the browser back to the root page (`/`) to keep the control panel open. |

## Hardware & Safety Concept: Non-Blocking Web Interactions and Sound Patterns
Buzzer warning patterns require turning the pin HIGH and LOW with precise delays (e.g. `delay(80)`). While this is blocking, keeping these durations short (under 500 ms) ensures the web server remains responsive to new requests. For longer alarms (like a continuous siren), manage the alarm state using non-blocking timers (Project 29).

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a warning screen when a chime is active.
2. **Frequency Selector**: Add a route `/freq?val=1000` to play a custom frequency tone on a passive buzzer.
3. **RSSI Signal warning**: Sound a warning beep if the WiFi connection drops.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the active buzzer is connected to GPIO 4 |
| The sound is squeaky or weak | Low drive current | Verify the buzzer pin is configured as an output |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Web-based LED toggle](44-esp32-web-based-led-toggle.md)
- [46 - ESP32 Web-based Relay switch control](46-esp32-web-based-relay-switch-control.md)
- [25 - ESP32 Wireless buzzer receiver](25-esp32-wireless-buzzer-receiver.md)
