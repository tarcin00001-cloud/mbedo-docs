# 111 - Rotary Encoder Web Menu Selector

Build a hybrid control console on the ESP32 that navigates a local 3-item menu on a 16x2 I2C LCD using a physical rotary encoder (CLK on GPIO 14, DT on GPIO 12, SW on GPIO 13), hosts a web dashboard displaying the active settings, and synchronizes changes bidirectionally between the web page and the hardware.

## Goal
Learn how to read rotary encoder rotation quadrature signals, debounce pushbutton switch inputs, manage menu navigation states on I2C LCDs, serve interactive web dashboards, and sync states bidirectionally.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A rotary encoder is on GPIO 14 (CLK), 12 (DT), and 13 (SW). A 16x2 LCD is on I2C (GPIO 21/22). The LCD shows a menu: Light (Auto/ON/OFF), Fan (Low/Med/High), and Reset. Rotating the encoder scrolls the menu, and pressing it confirms selections. Navigating to the ESP32's IP address displays the current settings. Clicking buttons on the webpage updates the device and LCD instantly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Rotary Encoder (KY-040) | `potentiometer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, use two pushbuttons (CLK/DT) or a potentiometer mapped to step values to simulate the rotary encoder's step outputs. Use a separate pushbutton (active-LOW) for the SW click button.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Rotary Encoder | CLK (Clock) | GPIO14 | Yellow | Quadrature pulse input |
| Rotary Encoder | DT (Data) | GPIO12 | Green | Direction state input |
| Rotary Encoder | SW (Switch) | GPIO13 | Blue | Select button trigger |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the encoder and LCD from the 5V Vin rail.

## Code
```cpp
// Rotary Encoder Web Menu Selector (Bidirectional Menu Console Sync)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int CLK_PIN = 14;
const int DT_PIN = 12;
const int SW_PIN = 13;

WebServer server(80);

// Menu State Variables
int menuIndex = 0;
const int MENU_SIZE = 3;
String menuItems[MENU_SIZE] = {"1. Light: ", "2. Fan Speed: ", "3. Reset Settings"};

// Sub-menu selections
String lightModes[3] = {"AUTO", "ON", "OFF"};
int activeLightMode = 0; // 0=AUTO, 1=ON, 2=OFF

String fanSpeeds[3] = {"LOW", "MED", "HIGH"};
int activeFanSpeed = 0; // 0=LOW, 1=MED, 2=HIGH

// Rotary Encoder Reading State
int lastCLKState;

// Debounce variables
unsigned long lastButtonPress = 0;

void updateLCDDisplay() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("-> Menu Selector");
  
  lcd.setCursor(0, 1);
  if (menuIndex == 0) {
    lcd.print(menuItems[0]);
    lcd.print(lightModes[activeLightMode]);
  } else if (menuIndex == 1) {
    lcd.print(menuItems[1]);
    lcd.print(fanSpeeds[activeFanSpeed]);
  } else if (menuIndex == 2) {
    lcd.print(menuItems[2]);
  }
}

// HTTP API endpoint returning JSON data
void handleGetMenuData() {
  String json = "{\"menuIdx\":" + String(menuIndex) + 
                 ",\"lightMode\":\"" + lightModes[activeLightMode] + "\"" +
                 ",\"fanSpeed\":\"" + fanSpeeds[activeFanSpeed] + "\"}";
  server.send(200, "application/json", json);
}

// HTTP POST endpoint to change settings from web
void handleSetMenuData() {
  if (server.hasArg("light")) {
    String lightVal = server.arg("light");
    for (int i = 0; i < 3; i++) {
      if (lightModes[i] == lightVal) activeLightMode = i;
    }
  }
  if (server.hasArg("fan")) {
    String fanVal = server.arg("fan");
    for (int i = 0; i < 3; i++) {
      if (fanSpeeds[i] == fanVal) activeFanSpeed = i;
    }
  }
  if (server.hasArg("reset")) {
    activeLightMode = 0;
    activeFanSpeed = 0;
    Serial.println("[Menu] Reset triggered via web.");
  }
  
  updateLCDDisplay();
  server.send(200, "text/plain", "OK");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Smart Console Dashboard</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 12px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; }\n";
  html += "  .btn-group { display: flex; gap: 10px; margin-top: 15px; }\n";
  html += "  .btn { flex-grow: 1; padding: 12px; font-size: 14px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn-primary { background-color: #38bdf8; border-color: #0ea5e9; color: #0f172a; }\n";
  html += "  .btn-primary:hover { background-color: #0ea5e9; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Smart Menu Console</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Light Mode</div><div class=\"metric-val\" id=\"lightDisplay\">AUTO</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Fan Speed</div><div class=\"metric-val\" id=\"fanDisplay\">LOW</div></div>\n";
  html += "  </div>\n";
  
  html += "  <h3>Web Adjustments</h3>\n";
  html += "  <div class=\"btn-group\">\n";
  html += "    <button class=\"btn\" onclick=\"setLight()\">Cycle Light</button>\n";
  html += "    <button class=\"btn\" onclick=\"setFan()\">Cycle Fan</button>\n";
  html += "    <button class=\"btn btn-primary\" onclick=\"resetSettings()\">Reset All</button>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 1 second | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  let currentLight = 'AUTO';\n";
  html += "  let currentFan = 'LOW';\n";
  
  html += "  function updateMenu() {\n";
  html += "    fetch('/api/menu')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        currentLight = data.lightMode;\n";
  html += "        currentFan = data.fanSpeed;\n";
  html += "        document.getElementById('lightDisplay').innerText = data.lightMode;\n";
  html += "        document.getElementById('fanDisplay').innerText = data.fanSpeed;\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  function setLight() {\n";
  html += "    const next = currentLight === 'AUTO' ? 'ON' : (currentLight === 'ON' ? 'OFF' : 'AUTO');\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('light', next);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body }).then(() => updateMenu());\n";
  html += "  }\n";
  
  html += "  function setFan() {\n";
  html += "    const next = currentFan === 'LOW' ? 'MED' : (currentFan === 'MED' ? 'HIGH' : 'LOW');\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('fan', next);\n";
  html += "    fetch('/api/control', { method: 'POST', body: body }).then(() => updateMenu());\n";
  html += "  }\n";
  
  html += "  function resetSettings() {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('reset', '1');\n";
  html += "    fetch('/api/control', { method: 'POST', body: body }).then(() => updateMenu());\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateMenu();\n";
  html += "    setInterval(updateMenu, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure Rotary Encoder Pins
  pinMode(CLK_PIN, INPUT);
  pinMode(DT_PIN, INPUT);
  pinMode(SW_PIN, INPUT_PULLUP);
  
  // Read initial CLK state
  lastCLKState = digitalRead(CLK_PIN);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  
  Serial.println("\nESP32 Rotary Encoder Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/menu", handleGetMenuData);
  server.on("/api/control", HTTP_POST, handleSetMenuData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  
  // 1. Read CLK rotation state
  int currentCLKState = digitalRead(CLK_PIN);
  
  // Check if CLK has transitioned (quadrature pulse)
  if (currentCLKState != lastCLKState && currentCLKState == LOW) {
    // If the DT state is different than CLK, rotation is clockwise
    if (digitalRead(DT_PIN) != currentCLKState) {
      menuIndex++;
      if (menuIndex >= MENU_SIZE) menuIndex = 0;
    } else {
      // Otherwise rotation is counter-clockwise
      menuIndex--;
      if (menuIndex < 0) menuIndex = MENU_SIZE - 1;
    }
    
    Serial.printf("[Encoder] Scrolled index: %d\n", menuIndex);
    updateLCDDisplay();
  }
  lastCLKState = currentCLKState;
  
  // 2. Read Switch Button SW pin state (active-LOW)
  if (digitalRead(SW_PIN) == LOW) {
    unsigned long now = millis();
    if (now - lastButtonPress >= 250) { // Debounce timer: 250 ms
      lastButtonPress = now;
      
      // Perform selection action
      if (menuIndex == 0) {
        // Cycle Light Mode selection
        activeLightMode = (activeLightMode + 1) % 3;
        Serial.printf("[Menu Action] Light Mode cycled: %s\n", lightModes[activeLightMode].c_str());
      } 
      else if (menuIndex == 1) {
        // Cycle Fan Speed selection
        activeFanSpeed = (activeFanSpeed + 1) % 3;
        Serial.printf("[Menu Action] Fan Speed cycled: %s\n", fanSpeeds[activeFanSpeed].c_str());
      } 
      else if (menuIndex == 2) {
        // Reset settings selection
        activeLightMode = 0;
        activeFanSpeed = 0;
        Serial.println("[Menu Action] Reset settings executed.");
      }
      
      updateLCDDisplay();
    }
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the CLK/DT rotation), **Pushbutton** (configured as active-LOW for SW), and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer output to **GPIO14**, SW Button to **GPIO13**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In hardware, turn the encoder to scroll through the menu, and press the switch button to cycle options. Verify that the webpage dashboard reflects the changes.
6. Click "Cycle Light" or "Cycle Fan" on the webpage. Verify that the LCD screen updates instantly.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Encoder] Scrolled index: 1
[Menu Action] Fan Speed cycled: MED
```

## Expected Canvas Behavior
* Pressing the simulated SW button cycles the selected menu item parameter on the LCD widget.
* Clicking web override buttons updates the display parameters on the LCD widget in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `digitalRead(CLK_PIN)` | Samples the Clock pin of the encoder to detect rotation transitions. |
| `digitalRead(DT_PIN) != currentCLKState` | Standard quadrature logic check to determine rotation direction. |
| `now - lastButtonPress >= 250` | Software debounce check to prevent double triggers on switch click. |
| `server.arg("light")` | Processes setting overrides sent via web console POST requests. |

## Hardware & Safety Concept: Rotary Encoder Quadrature Logic and Debounce Filtering
* **Quadrature Logic**: Rotary encoders use two internal switches (CLK and DT) offset by 90 degrees. Tapping them generates square waves. By comparing which wave transitions first, the microcontroller detects rotation direction.
* **Debouncing SW**: Mechanical switches bounce when clicked, triggering multiple fake presses. Implementing a software lockout interval (e.g. 250 ms) ignores bounce noise, ensuring smooth selector inputs.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and design a styled circular dial menu.
2. **Alert buzzer tone**: Sound a brief click tone on a buzzer (GPIO 15) when the encoder scrolls or is clicked.
3. **SPIFFS configuration**: Store menu parameters in the SPIFFS filesystem (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Encoder scrolls erratically | Quadrature signals bouncing | Add 0.1 µF bypass capacitors across CLK/DT pins to ground to filter noise |
| Select button does not work | Pull-up missing | Verify that the SW pin is configured with `INPUT_PULLUP` |
| LCD does not print text | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [101 - ESP32 Automatic Barrier Gate IoT Console](101-esp32-automatic-barrier-gate-iot-console.md)
- [112 - ESP32 DS3231 Real Time Clock Web Sync Display](112-esp32-ds3231-real-time-clock-web-sync-display.md) (Next project)
