# 54 - Web Page Relay Slider Switch

Build a responsive home automation control page on the ESP32 that renders a custom CSS sliding toggle switch (similar to an iOS toggle), processes state updates, and drives a physical relay connected to GPIO 13.

## Goal
Learn how to create custom CSS slider toggles, handle state changes, manage HTML structures, and interface with relays.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A relay is on GPIO 13. Navigating to the ESP32's IP address displays a webpage with a sliding toggle switch. Clicking the slider toggles the relay and updates the dashboard.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load controller switch |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the relay from the 5V Vin rail.

## Code
```cpp
// Web Page Relay slider switch (Styled sliding toggle controller)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int RELAY_PIN = 13;

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  bool relayState = (digitalRead(RELAY_PIN) == HIGH);
  
  // Compile HTML page with a modern CSS sliding switch
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Relay Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 320px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 20px; color: #38bdf8; margin-top: 0; margin-bottom: 30px; }\n";
  
  // Custom CSS Sliding Switch styling
  html += "  .switch { position: relative; display: inline-block; width: 80px; height: 44px; }\n";
  html += "  .switch input { opacity: 0; width: 0; height: 0; }\n";
  html += "  .slider { position: absolute; cursor: pointer; top: 0; left: 0; right: 0; bottom: 0; background-color: #475569; transition: .4s; border-radius: 34px; }\n";
  html += "  .slider:before { position: absolute; content: \"\"; height: 36px; width: 36px; left: 4px; bottom: 4px; background-color: white; transition: .4s; border-radius: 50%; }\n";
  html += "  input:checked + .slider { background-color: #10b981; }\n";
  html += "  input:checked + .slider:before { transform: translateX(36px); }\n";
  
  html += "  .status-label { margin-top: 25px; font-size: 18px; font-weight: 500; color: #94a3b8; }\n";
  html += "  .status-active { color: #10b981; font-weight: bold; }\n";
  html += "  .status-inactive { color: #ef4444; font-weight: bold; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Appliance Relay</h1>\n";
  
  // Checkbox slider input triggering redirection on click event
  html += "  <label class=\"switch\">\n";
  html += "    <input type=\"checkbox\" id=\"relaySlider\" onchange=\"toggleRelay()\" " + String(relayState ? "checked" : "") + ">\n";
  html += "    <span class=\"slider\"></span>\n";
  html += "  </label>\n";
  
  html += "  <div class=\"status-label\">State: ";
  if (relayState) {
    html += "<span class=\"status-active\">ON (CLOSED)</span>";
  } else {
    html += "<span class=\"status-inactive\">OFF (OPEN)</span>";
  }
  html += "  </div>\n";
  html += "</div>\n";
  
  // Inline JavaScript to trigger redirection
  html += "<script>\n";
  html += "  function toggleRelay() {\n";
  html += "    window.location.href = '/toggle';\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Toggle URL Handler ("/toggle")
void handleToggle() {
  bool nextState = !digitalRead(RELAY_PIN);
  digitalWrite(RELAY_PIN, nextState ? HIGH : LOW);
  
  Serial.printf("[Server] Slider action processed. Relay is now: %s\n", nextState ? "CLOSED (ON)" : "OPEN (OFF)");
  
  // Redirect back to root page to refresh state
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with relay OPEN (off)
  
  Serial.println("\nESP32 Smart Relay Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/toggle", handleToggle);
  
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
1. Drag **ESP32 DevKitC** and **Relay** onto the canvas.
2. Wire Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Click the slider to toggle the relay.

## Expected Output
Serial Monitor:
```
ESP32 Smart Relay Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Slider action processed. Relay is now: CLOSED (ON)
[Server] Slider action processed. Relay is now: OPEN (OFF)
```

## Expected Canvas Behavior
* Clicking the slider on the web page toggles the relay widget ON (green) and OFF (grey).
* The Serial Monitor logs the client requests.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `input:checked + .slider` | Selects the slider element when the checkbox is checked, changing the background color to green. |
| `window.location.href = '/toggle'` | Redirects the browser to `/toggle` when the checkbox state changes. |
| `digitalWrite(RELAY_PIN, ...)` | Controls the relay contacts to switch the connected load. |

## Hardware & Safety Concept: Slider Interaction and Debouncing
Sliding toggle controls improve the user interface. When implementing them on web pages:
1. **Interactive Event Triggers**: The javascript handler `onchange` redirects the browser immediately when clicked, keeping the interface responsive.
2. **Contact Protection**: Limit the frequency of toggle transitions to prevent the relay from switching rapidly (chattering) if the user clicks the button repeatedly.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current relay status.
2. **AJAX Auto-toggle**: Modify the code to toggle the state using AJAX (Project 60) instead of reloading the entire page.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when the relay state changes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Clicking the slider returns 404 | URL route mismatch | Verify that the `/toggle` route is registered in setup |
| Relay does not switch | Pin assignment conflict | Verify that the relay signal line is wired to GPIO 13 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [46 - ESP32 Web-based Relay switch control](../beginner/46-esp32-web-based-relay-switch-control.md)
- [53 - ESP32 Web Page Multi-LED Control panel](53-esp32-web-page-multi-led-control-panel.md)
- [55 - ESP32 Web Page Buzzer frequency selector](55-esp32-web-page-buzzer-frequency-selector.md) (Next project)
