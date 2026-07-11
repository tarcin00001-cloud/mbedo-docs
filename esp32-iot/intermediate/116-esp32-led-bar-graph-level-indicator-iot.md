# 116 - LED Bar Graph Level Indicator IoT (Web Slider -> Bar Graph)

Build an interactive indicator system on the ESP32 that maps a web-based slider level (0 to 5) to a physical LED bar graph composed of 5 LEDs on GPIOs 12, 14, 27, 26, and 25.

## Goal
Learn how to define pin arrays, implement loop-based state maps, serve interactive slider consoles, and process HTTP POST level updates.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Five LEDs are connected to GPIOs 12, 14, 27, 26, and 25 to form a bar graph indicator. Navigating to the ESP32's IP address displays a webpage with an interactive level slider (0 to 5). Dragging the slider sends updates to the ESP32, which lights up the matching number of LEDs (e.g. level 3 turns ON the first three LEDs and turns OFF the rest).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LEDs (Red/Yellow/Green) | `led` | Yes (5 pins) | Yes (5 pieces) |
| 220 Ω Resistors | `resistor` | No | Yes (5 pieces) |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED 1 | Anode (+) | GPIO12 | Red | Level 1 indicator |
| LED 2 | Anode (+) | GPIO14 | Yellow | Level 2 indicator |
| LED 3 | Anode (+) | GPIO27 | Green | Level 3 indicator |
| LED 4 | Anode (+) | GPIO26 | Blue | Level 4 indicator |
| LED 5 | Anode (+) | GPIO25 | White | Level 5 indicator |
| Shared Resistors | Cathode (-) | GND | Black | Current limiting |

> **Wiring tip:** Connect a 220 Ω resistor in series with each LED's cathode pin to limit current and protect the ESP32 pins.

## Code
```cpp
// LED Bar Graph Level Indicator IoT (Web Slider Control Console)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Array of GPIO pins connected to the LEDs
const int NUM_LEDS = 5;
const int ledPins[NUM_LEDS] = {12, 14, 27, 26, 25};

WebServer server(80);
int activeLevel = 0; // Ranges from 0 to 5

// Apply active level to physical LEDs
void updateLEDBarGraph() {
  for (int i = 0; i < NUM_LEDS; i++) {
    if (i < activeLevel) {
      digitalWrite(ledPins[i], HIGH);  // Turn ON LED
    } else {
      digitalWrite(ledPins[i], LOW);   // Turn OFF LED
    }
  }
  Serial.printf("[Bar Graph] Level set to %d / %d LEDs active.\n", activeLevel, NUM_LEDS);
}

// HTTP API endpoint returning JSON data
void handleGetLevelData() {
  String json = "{\"level\":" + String(activeLevel) + "}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to update level
void handlePostLevel() {
  if (server.hasArg("level")) {
    activeLevel = server.arg("level").toInt();
    
    // Bounds check
    if (activeLevel < 0) activeLevel = 0;
    if (activeLevel > NUM_LEDS) activeLevel = NUM_LEDS;
    
    updateLEDBarGraph();
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>LED Level Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  
  // Led bar representation
  html += "  .bar-container { display: flex; gap: 8px; justify-content: center; margin: 30px 0; }\n";
  html += "  .led-segment { width: 30px; height: 15px; border-radius: 4px; background-color: #334155; border: 1px solid #475569; transition: background-color 0.1s; }\n";
  html += "  .led-segment.active-0 { background-color: #ef4444; box-shadow: 0 0 10px #ef4444; } /* Red */\n";
  html += "  .led-segment.active-1 { background-color: #f97316; box-shadow: 0 0 10px #f97316; } /* Orange */\n";
  html += "  .led-segment.active-2 { background-color: #eab308; box-shadow: 0 0 10px #eab308; } /* Yellow */\n";
  html += "  .led-segment.active-3 { background-color: #22c55e; box-shadow: 0 0 10px #22c55e; } /* Green */\n";
  html += "  .led-segment.active-4 { background-color: #3b82f6; box-shadow: 0 0 10px #3b82f6; } /* Blue */\n";
  
  html += "  .slider-container { margin: 25px 0; text-align: left; }\n";
  html += "  label { font-weight: bold; color: #94a3b8; display: block; margin-bottom: 12px; }\n";
  html += "  input[type=range] { width: 100%; height: 8px; border-radius: 4px; background: #334155; outline: none; -webkit-appearance: none; }\n";
  html += "  input[type=range]::-webkit-slider-thumb { -webkit-appearance: none; width: 22px; height: 22px; border-radius: 50%; background: #38bdf8; cursor: pointer; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>LED Level Controller</h1>\n";
  
  // LED segment display representation
  html += "  <div class=\"bar-container\">\n";
  for (int i = 0; i < NUM_LEDS; i++) {
    html += "    <div id=\"seg" + String(i) + "\" class=\"led-segment\"></div>\n";
  }
  html += "  </div>\n";
  
  html += "  <div class=\"slider-container\">\n";
  html += "    <label for=\"lvlRange\">Active Level: <span id=\"lvlVal\">0</span> / 5</label>\n";
  html += "    <input type=\"range\" id=\"lvlRange\" min=\"0\" max=\"5\" value=\"" + String(activeLevel) + "\" oninput=\"updateLevel(this.value)\">\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateLevelDisplay() {\n";
  html += "    fetch('/api/level')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('lvlRange').value = data.level;\n";
  html += "        document.getElementById('lvlVal').innerText = data.level;\n";
  
  html += "        for (let i = 0; i < 5; i++) {\n";
  html += "          const seg = document.getElementById('seg' + i);\n";
  html += "          if (i < data.level) {\n";
  html += "            seg.className = 'led-segment active-' + i;\n";
  html += "          } else {\n";
  html += "            seg.className = 'led-segment';\n";
  html += "          }\n";
  html += "        }\n";
  html += "      });\n";
  // Sync page segments initially
  html += "  }\n";
  
  html += "  function updateLevel(val) {\n";
  html += "    document.getElementById('lvlVal').innerText = val;\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('level', val);\n";
  html += "    fetch('/api/level', { method: 'POST', body: body })\n";
  html += "      .then(() => updateLevelDisplay());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateLevelDisplay();\n";
  html += "    setInterval(updateLevelDisplay, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LED Pins
  for (int i = 0; i < NUM_LEDS; i++) {
    pinMode(ledPins[i], OUTPUT);
    digitalWrite(ledPins[i], LOW); // Start with all LEDs off
  }
  
  Serial.println("\nESP32 LED Bar Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/level", HTTP_GET, handleGetLevelData);
  server.on("/api/level", HTTP_POST, handlePostLevel);
  
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
1. Drag **ESP32 DevKitC** and **5 LEDs** (Red, Orange, Yellow, Green, Blue) onto the canvas.
2. Wire the LED anodes to **GPIO12, 14, 27, 26, and 25** respectively. Wire the cathodes to GND.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Drag the level slider on the webpage. Verify that the LEDs on the canvas light up sequentially to match the selected level.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Bar Graph] Level set to 3 / 5 LEDs active.
```

## Expected Canvas Behavior
* Dragging the slider on the webpage lights up the corresponding number of LED widgets on the canvas instantly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledPins[i]` | Accesses the GPIO pin number mapped to that segment index. |
| `i < activeLevel` | Determines if the segment should be turned ON or OFF. |
| `server.arg("level")` | Extracts the level parameter value sent via HTTP POST requests. |

## Hardware & Safety Concept: Current Limitations and LED Multiplexing
* **Current limits**: Each ESP32 GPIO pin can source or sink a maximum of 40 mA of current, with a total package limit of 1200 mA. Lighting up multiple LEDs simultaneously draws significant current. Always use current-limiting resistors (e.g. 220 Ω or 330 Ω) to keep current per pin under 10 mA.
* **Driver ICs**: For larger LED bar graphs (e.g. 10 or more segments), use dedicated driver ICs like the **LM3914** or shift registers like the **74HC595** to control the LEDs using only 3 control pins.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a matching visual level dial.
2. **Alert Siren**: Add a buzzer (GPIO 15) and sound an alarm if the level reaches 5 (critical limit).
3. **SPIFFS integration**: Save the last selected level in SPIFFS (Project 62) so the scale restores its setting on reboot.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LEDs do not light up | Cathodes not grounded | Ensure that all LED cathode pins are connected to a common ground rail |
| LEDs light up in the wrong order | Pin mapping swapped | Verify that the wiring matches the order defined in the `ledPins` array |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [04 - Dual-LED alternating flasher](../beginner/04-dual-led-alternating-flasher.md)
- [49 - Potentiometer LED Bar Graph level controller](../beginner/49-potentiometer-led-bar-graph-level-controller.md)
- [117 - ESP32 IR Remote Command Web Decoder](117-esp32-ir-remote-command-web-decoder.md) (Next project)
