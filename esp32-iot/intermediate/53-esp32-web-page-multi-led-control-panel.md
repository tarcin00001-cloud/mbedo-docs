# 53 - Web Page Multi-LED Control Panel

Build a multi-device web control panel on the ESP32 that hosts a local dashboard, processes query parameters to identify target pins (`/toggle?pin=13`), and toggles Red, Green, and Yellow LEDs independently.

## Goal
Learn how to parse HTTP query parameters, control multiple digital pins independently, build interactive CSS grid dashboards, and manage redirects.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. Three LEDs are connected to GPIO 13 (Red), GPIO 12 (Green), and GPIO 14 (Yellow). Navigating to the ESP32's IP address displays a dashboard containing three cards with toggle buttons. Clicking any button sends a parameter-matched request to `/toggle?pin=X`, toggling that specific LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| LED (Yellow) | `led` | Yes | Yes |
| 3x 330 Ω Resistors | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Channel 1 indicator |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Channel 2 indicator |
| Yellow LED | Anode (+) | GPIO14 via 330 Ω | Yellow | Channel 3 indicator |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect each LED anode to its respective GPIO pin via a 330 Ω resistor.

## Code
```cpp
// Web Page Multi-LED Control panel (Parameter-driven pin switching)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configure LED Pin assignments
const int RED_LED = 13;
const int GREEN_LED = 12;
const int YELLOW_LED = 14;

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  bool redState = (digitalRead(RED_LED) == HIGH);
  bool greenState = (digitalRead(GREEN_LED) == HIGH);
  bool yellowState = (digitalRead(YELLOW_LED) == HIGH);
  
  // Compile HTML page with a styled responsive grid dashboard
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Multi-LED Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .container { max-width: 800px; width: 100%; }\n";
  html += "  h1 { font-size: 26px; color: #38bdf8; text-align: center; margin-bottom: 30px; }\n";
  html += "  .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px, 1fr)); gap: 20px; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 20px; text-align: center; border: 1px solid #334155; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.3); }\n";
  html += "  .card-red h2 { color: #f43f5e; } .card-green h2 { color: #10b981; } .card-yellow h2 { color: #eab308; }\n";
  html += "  .status { font-size: 16px; margin: 15px 0 25px 0; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 12px; font-size: 15px; font-weight: bold; color: white; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; text-align: center; box-sizing: border-box; }\n";
  html += "  .btn-on { background-color: #059669; }\n";
  html += "  .btn-off { background-color: #dc2626; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"container\">\n";
  html += "  <h1>Multi-Channel Web Control Panel</h1>\n";
  html += "  <div class=\"grid\">\n";
  
  // Card 1: Red LED
  html += "    <div class=\"card card-red\">\n";
  html += "      <h2>Red Channel</h2>\n";
  html += "      <p class=\"status\">State: <span class=\"status-val\" style=\"color: " + String(redState ? "#f43f5e" : "#64748b") + "\">" + String(redState ? "ACTIVE" : "INACTIVE") + "</span></p>\n";
  html += "      <a href=\"/toggle?pin=" + String(RED_LED) + "\" class=\"btn " + String(redState ? "btn-off" : "btn-on") + "\">" + String(redState ? "TURN OFF" : "TURN ON") + "</a>\n";
  html += "    </div>\n";
  
  // Card 2: Green LED
  html += "    <div class=\"card card-green\">\n";
  html += "      <h2>Green Channel</h2>\n";
  html += "      <p class=\"status\">State: <span class=\"status-val\" style=\"color: " + String(greenState ? "#10b981" : "#64748b") + "\">" + String(greenState ? "ACTIVE" : "INACTIVE") + "</span></p>\n";
  html += "      <a href=\"/toggle?pin=" + String(GREEN_LED) + "\" class=\"btn " + String(greenState ? "btn-off" : "btn-on") + "\">" + String(greenState ? "TURN OFF" : "TURN ON") + "</a>\n";
  html += "    </div>\n";
  
  // Card 3: Yellow LED
  html += "    <div class=\"card card-yellow\">\n";
  html += "      <h2>Yellow Channel</h2>\n";
  html += "      <p class=\"status\">State: <span class=\"status-val\" style=\"color: " + String(yellowState ? "#eab308" : "#64748b") + "\">" + String(yellowState ? "ACTIVE" : "INACTIVE") + "</span></p>\n";
  html += "      <a href=\"/toggle?pin=" + String(YELLOW_LED) + "\" class=\"btn " + String(yellowState ? "btn-off" : "btn-on") + "\">" + String(yellowState ? "TURN OFF" : "TURN ON") + "</a>\n";
  html += "    </div>\n";
  
  html += "  </div>\n";
  html += "</div>\n";
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Toggle URL Query Parameter Handler ("/toggle")
void handleToggle() {
  // 1. Verify if query parameter "pin" is present
  if (server.hasArg("pin")) {
    int targetPin = server.arg("pin").toInt();
    
    // 2. Security validation: Ensure target pin is one of our configured channels
    if (targetPin == RED_LED || targetPin == GREEN_LED || targetPin == YELLOW_LED) {
      bool currentState = digitalRead(targetPin);
      digitalWrite(targetPin, !currentState);
      Serial.printf("[Server] Parameter Toggle on pin %d: Next State -> %s\n", targetPin, !currentState ? "ON" : "OFF");
    } else {
      Serial.printf("[Server Error] Invalid pin access attempted: %d\n", targetPin);
    }
  }
  
  // Redirect back to root page to update display status
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RED_LED, OUTPUT);
  pinMode(GREEN_LED, OUTPUT);
  pinMode(YELLOW_LED, OUTPUT);
  
  digitalWrite(RED_LED, LOW);
  digitalWrite(GREEN_LED, LOW);
  digitalWrite(YELLOW_LED, LOW);
  
  Serial.println("\nESP32 Multi-LED Web Server Starting...");
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
1. Drag **ESP32 DevKitC** and **3 LEDs** (Red, Green, Yellow) onto the canvas.
2. Wire LEDs to **GPIO13** (Red), **GPIO12** (Green), and **GPIO14** (Yellow).
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Click the buttons to control the LEDs independently.

## Expected Output
Serial Monitor:
```
ESP32 Multi-LED Web Server Starting...
....
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Parameter Toggle on pin 13: Next State -> ON
[Server] Parameter Toggle on pin 12: Next State -> ON
[Server] Parameter Toggle on pin 13: Next State -> OFF
```

## Expected Canvas Behavior
* Clicking the buttons on the web page controls the Red, Green, and Yellow LED widgets independently.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `server.hasArg("pin")` | Checks if the HTTP request contains the `pin` query argument. |
| `server.arg("pin").toInt()` | Extracts and converts the argument string to an integer pin address. |
| `targetPin == RED_LED || ...` | Validates that the requested pin matches one of our configured channels before modifying the state. |
| `server.send(303)` | Redirects the browser back to `/` to refresh the dashboard status. |

## Hardware & Safety Concept: Parameter Validation and Security in Web Control
When designing APIs that control physical hardware, **always validate input parameters**. If a web server executes commands on any pin passed in a query (e.g. `digitalWrite(targetPin, state)`), a malicious request could switch critical internal pins (such as flash communication pins or reset pins), crashing the device or causing damage. Enforcing whitelist validation prevents this.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing which LEDs are active.
2. **Master Control Button**: Add a button to turn all three channels ON or OFF simultaneously.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 15) when any channel switches.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The webpage fails to load | Client on different network | Ensure your computer and the ESP32 are connected to the same WiFi network |
| Clicking a button returns 404 | URL route parameter mismatch | Verify that the link is formatted correctly as `/toggle?pin=X` |
| One LED does not light | Pin assignment mismatch | Confirm that the LED is wired to the correct GPIO pin (13, 12, or 14) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [44 - ESP32 Web-based LED toggle](../beginner/44-esp32-web-based-led-toggle.md)
- [52 - ESP32 Web Page LED Toggle button](52-esp32-web-page-led-toggle-button.md)
- [54 - ESP32 Web Page Relay slider switch](54-esp32-web-page-relay-slider-switch.md) (Next project)
