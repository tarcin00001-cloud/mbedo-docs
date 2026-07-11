# 118 - IR Remote Web LED Control

Build a hybrid control system on the ESP32 that toggles local LEDs (Red on GPIO 12, Green on GPIO 13) using a handheld infrared remote control via an IR receiver on GPIO 14, and synchronizes the LED states in real time with a web control panel.

## Goal
Learn how to decode infrared commands using `IRremote`, map hex key codes to specific logic branches, build interactive web controls, and synchronize state updates bidirectionally.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An IR receiver is on GPIO 14, a Red LED on GPIO 12, and a Green LED on GPIO 13. Pressing button '1' on the IR remote toggles the Red LED, and button '2' toggles the Green LED. Navigating to the ESP32's IP address displays a control panel showing the current state of both LEDs and allows toggling them via buttons on the webpage.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Receiver Module (TSOP38238) | `ir_receiver` | Yes | Yes |
| Handheld IR Remote | `ir_remote` | Yes | Yes |
| Red LED / Green LED | `led` | Yes | Yes |
| 220 Ω Resistors | `resistor` | No | Yes (2 pieces) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | OUT (Signal) | GPIO14 | Yellow | IR pulse decoder input |
| Red LED | Anode (+) | GPIO12 | Red | Red load indicator |
| Green LED | Anode (+) | GPIO13 | Green | Green load indicator |
| Shared Resistors | Cathode (-) | GND | Black | Current limiting |

> **Wiring tip:** Power the IR receiver from the 3.3V pin. Connect a 220 Ω resistor in series with each LED.

## Code
```cpp
// IR Remote Web LED Control (Infrared remote toggle + Web state sync)
#define DECODE_NEC // Enable NEC decoding protocol
#include <WiFi.h>
#include <WebServer.h>
#include <IRremote.hpp>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int IR_RECEIVE_PIN = 14;
const int RED_LED_PIN = 12;
const int GRN_LED_PIN = 13;

// Map target remote control buttons (NEC commands)
// Adjust these commands if using a different IR remote
const uint16_t CMD_RED_TOGGLE = 0x0C; // Button '1' on standard remote
const uint16_t CMD_GRN_TOGGLE = 0x18; // Button '2' on standard remote

WebServer server(80);

bool redState = false;
bool grnState = false;

// Toggle Red LED state
void toggleRed() {
  redState = !redState;
  digitalWrite(RED_LED_PIN, redState ? HIGH : LOW);
  Serial.printf("[Control] Red LED toggled -> %s\n", redState ? "ON" : "OFF");
}

// Toggle Green LED state
void toggleGreen() {
  grnState = !grnState;
  digitalWrite(GRN_LED_PIN, grnState ? HIGH : LOW);
  Serial.printf("[Control] Green LED toggled -> %s\n", grnState ? "ON" : "OFF");
}

// HTTP API endpoint returning JSON data
void handleGetLEDs() {
  String json = "{\"red\":" + String(redState ? 1 : 0) + 
                 ",\"green\":" + String(grnState ? 1 : 0) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to control LEDs from web
void handlePostControl() {
  if (server.hasArg("target")) {
    String target = server.arg("target");
    if (target == "red") {
      toggleRed();
    } else if (target == "green") {
      toggleGreen();
    }
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Control Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .control-row { display: flex; justify-content: space-around; margin: 30px 0; }\n";
  html += "  .led-widget { display: flex; flex-direction: column; align-items: center; gap: 10px; }\n";
  html += "  .indicator { width: 40px; height: 40px; border-radius: 50%; background-color: #334155; border: 2px solid #475569; transition: background-color 0.2s, box-shadow 0.2s; }\n";
  html += "  .indicator.red-on { background-color: #ef4444; border-color: #f87171; box-shadow: 0 0 15px rgba(239, 68, 68, 0.5); }\n";
  html += "  .indicator.green-on { background-color: #22c55e; border-color: #4ade80; box-shadow: 0 0 15px rgba(34, 197, 94, 0.5); }\n";
  html += "  .btn { padding: 10px 20px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #475569; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>IR Web LED Controller</h1>\n";
  
  html += "  <div class=\"control-row\">\n";
  html += "    <div class=\"led-widget\">\n";
  html += "      <div id=\"redInd\" class=\"indicator\"></div>\n";
  html += "      <button class=\"btn\" onclick=\"toggleLED('red')\">Toggle Red</button>\n";
  html += "    </div>\n";
  html += "    <div class=\"led-widget\">\n";
  html += "      <div id=\"grnInd\" class=\"indicator\"></div>\n";
  html += "      <button class=\"btn\" onclick=\"toggleLED('green')\">Toggle Green</button>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateLEDStates() {\n";
  html += "    fetch('/api/leds')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('redInd').className = 'indicator' + (data.red === 1 ? ' red-on' : '');\n";
  html += "        document.getElementById('grnInd').className = 'indicator' + (data.green === 1 ? ' green-on' : '');\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function toggleLED(ledColor) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('target', ledColor);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body })\n";
  html += "      .then(() => updateLEDStates());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateLEDStates();\n";
  html += "    setInterval(updateLEDStates, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(RED_LED_PIN, OUTPUT);
  pinMode(GRN_LED_PIN, OUTPUT);
  
  digitalWrite(RED_LED_PIN, LOW); // Start with LEDs OFF
  digitalWrite(GRN_LED_PIN, LOW);
  
  // Initialize IR Receiver module
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
  
  Serial.println("\nESP32 IR Web control Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/leds", HTTP_GET, handleGetLEDs);
  server.on("/api/control", HTTP_POST, handlePostControl);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 1. Check for incoming IR signals
  if (IrReceiver.decode()) {
    uint16_t command = IrReceiver.decodedIRData.command;
    
    // Ignore repeat signals
    if (!(IrReceiver.decodedIRData.flags & IRDATA_FLAGS_IS_REPEAT)) {
      Serial.printf("[IR Input] Received command: 0x%04X\n", command);
      
      if (command == CMD_RED_TOGGLE) {
        toggleRed();
      } 
      else if (command == CMD_GRN_TOGGLE) {
        toggleGreen();
      }
    }
    
    // Resume IR receiver
    IrReceiver.resume();
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Receiver**, **IR Remote**, **Red LED**, and **Green LED** onto the canvas.
2. Wire the IR Receiver's output to **GPIO14**, Red LED to **GPIO12**, and Green LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click button `1` on the simulated IR Remote widget. Verify that the Red LED turns ON and the web dashboard updates.
6. Click the "Toggle Green" button on the webpage. Verify that the Green LED turns ON.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[IR Input] Received command: 0x000C
[Control] Red LED toggled -> ON
```

## Expected Canvas Behavior
* Pressing button `1` on the IR remote widget immediately lights up the Red LED widget on the canvas.
* Clicking web buttons toggles the LED widgets.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `command == CMD_RED_TOGGLE` | Maps the hex code for button `1` to toggle the Red LED. |
| `digitalWrite(RED_LED_PIN, ...)` | Sets the GPIO output pin HIGH/LOW to drive the LED. |
| `server.arg("target")` | Processes the targeted LED color parameter sent via web POST requests. |

## Hardware & Safety Concept: Command Deconfliction and State Synchronization
* **Command Deconfliction**: Different IR remotes use different protocols and key codes. When building systems, write a test sketch to print the hex values of your remote's buttons to serial first, then copy those values into your main code.
* **State Synchronization**: For home automation, the system must keep track of states in memory. If a user turns a light OFF using the remote, the web page status must update instantly to prevent UI mismatch.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a lightbulb icon indicating which LED is ON.
2. **PWM Dimming**: Replace digital toggle controls with PWM brightness levels using LEDC (Project 21) driven by the remote's up/down arrows.
3. **All Off Button**: Map a remote button (e.g. Power button) to turn off both LEDs simultaneously.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Remote buttons do not toggle LEDs | Key codes mismatch | Verify your remote's key codes by printing them to the Serial Monitor |
| LEDs do not light up | Cathodes not grounded | Confirm that the LED cathode pins are connected to a common ground rail |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [04 - Dual-LED alternating flasher](../beginner/04-dual-led-alternating-flasher.md)
- [117 - ESP32 IR Remote Command Web Decoder](117-esp32-ir-remote-command-web-decoder.md)
- [119 - ESP32 OLED Digital Web Stop Watch](119-esp32-oled-digital-web-stop-watch.md) (Next project)
