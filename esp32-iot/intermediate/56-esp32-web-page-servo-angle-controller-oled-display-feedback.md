# 56 - Web Page Servo Angle Controller (OLED display feedback)

Build a web-controlled positioning station on the ESP32 that hosts a control panel, parses target angle parameters from HTTP requests (`/set?angle=90`), drives a servo motor connected to GPIO 13, and displays status feedback on an SSD1306 OLED screen.

## Goal
Learn how to parse query parameters, control servo motor angles using `ESP32Servo`, update SSD1306 OLED screens via I2C, and handle redirects.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A servo motor is on GPIO 13, and an SSD1306 OLED on I2C (GPIO 21/22). Navigating to the ESP32's IP address displays a webpage with a slider (0° to 180°). Submitting an angle turns the servo shaft to that position and displays the value on the OLED screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Servo Motor | PWM (Signal) | GPIO13 | Orange | Servo signal line |
| SSD1306 OLED | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the servo from the 5V Vin rail.

## Code
```cpp
// Web Page Servo angle controller (Servo positioning + OLED status feedback)
#include <WiFi.h>
#include <WebServer.h>
#include <ESP32Servo.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
#define OLED_ADDR     0x3C

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Servo myServo;

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SERVO_PIN = 13;

// Active servo angle state variable
int activeAngle = 90; // Default center position

WebServer server(80);

void updateOLED() {
  display.clearDisplay();
  
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("SERVO CONTROLLER");
  display.drawFastHLine(0, 10, 128, SSD1306_WHITE);
  
  display.setCursor(0, 20);
  display.print("WiFi: Connected");
  display.setCursor(0, 32);
  display.print("IP: "); display.print(WiFi.localIP());
  
  // Large angle feedback
  display.setCursor(0, 48);
  display.setTextSize(2);
  display.print("Ang: "); display.print(activeAngle); display.write(247); // Degree symbol
  
  display.display();
}

// Root URL Route Handler ("/")
void handleRoot() {
  // Compile HTML page with angle slider controls
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Servo Angle Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 20px; }\n";
  html += "  .status { font-size: 18px; margin: 20px 0; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; color: #10b981; }\n";
  html += "  .slider { width: 100%; margin: 20px 0; }\n";
  html += "  .btn { display: inline-block; padding: 12px 24px; font-size: 15px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  .presets { display: flex; justify-content: space-between; margin-top: 20px; border-top: 1px solid #334155; padding-top: 20px; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Servo Angle Controller</h1>\n";
  
  // Status display
  html += "  <p class=\"status\">Current Angle: <span class=\"status-val\">" + String(activeAngle) + "&deg;</span></p>\n";
  
  // Slider form for custom angle positioning
  html += "  <form action=\"/set\" method=\"GET\">\n";
  html += "    <input type=\"range\" id=\"angleRange\" name=\"angle\" min=\"0\" max=\"180\" value=\"" + String(activeAngle) + "\" class=\"slider\" oninput=\"updateSliderVal(this.value)\">\n";
  html += "    <span id=\"sliderVal\">" + String(activeAngle) + "&deg;</span><br><br>\n";
  html += "    <button type=\"submit\" class=\"btn\">Position Servo</button>\n";
  html += "  </form>\n";
  
  // Presets
  html += "  <div class=\"presets\">\n";
  html += "    <a href=\"/set?angle=0\" class=\"btn\">0&deg;</a>\n";
  html += "    <a href=\"/set?angle=90\" class=\"btn\">90&deg;</a>\n";
  html += "    <a href=\"/set?angle=180\" class=\"btn\">180&deg;</a>\n";
  html += "  </div>\n";
  
  html += "</div>\n";
  
  // JS script to update text value dynamically as slider moves
  html += "<script>\n";
  html += "  function updateSliderVal(val) {\n";
  html += "    document.getElementById('sliderVal').innerHTML = val + '&deg;';\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Servo position setting handler ("/set")
void handleSet() {
  if (server.hasArg("angle")) {
    int angle = server.arg("angle").toInt();
    
    // Validate target angle within bounds
    if (angle >= 0 && angle <= 180) {
      activeAngle = angle;
      
      // Write target angle to servo motor
      myServo.write(activeAngle);
      Serial.printf("[Server] Servo angle adjusted: %d degrees\n", activeAngle);
      
      // Update OLED screen
      updateOLED();
    } else {
      Serial.printf("[Server Error] Out of bounds angle attempted: %d degrees\n", angle);
    }
  }
  
  // Redirect back to root page to update status
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize I2C OLED display
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Servo Web Control");
  display.println("Connecting WiFi...");
  display.display();
  
  // Configure and attach Servo motor
  myServo.attach(SERVO_PIN);
  myServo.write(activeAngle); // Center servo (90 degrees)
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Update initial OLED screen
  updateOLED();
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/set", handleSet);
  
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
1. Drag **ESP32 DevKitC**, **Servo**, and **SSD1306 OLED** onto the canvas.
2. Wire Servo to **GPIO13** and OLED to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the slider or click the preset buttons to position the servo and verify OLED display feedback.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Servo angle adjusted: 45 degrees
[Server] Servo angle adjusted: 135 degrees
```

OLED Display Layout:
```
SERVO CONTROLLER
───────────────────
WiFi: Connected
IP: 10.10.0.3
Ang: 135*
```

## Expected Canvas Behavior
* Adjusting the slider on the web page updates the servo motor position.
* The OLED widget updates the displayed angle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `myServo.attach(SERVO_PIN)` | Configures the ESP32 hardware timer to drive the servo signal pin. |
| `myServo.write(activeAngle)` | Sends the target angle command to the servo controller. |
| `updateOLED()` | Updates the OLED screen to display the current angle. |

## Hardware & Safety Concept: Servo Motor Power and Angular Boundaries
Servo motors draw significant current when moving (up to 500 mA). If powered directly from the ESP32's 3.3V rail, they can drop the voltage and reset the microcontroller. Always power the servo from the **5V Vin rail** or an external power supply. Additionally, validate that the angle is restricted between 0° and 180° to prevent the servo from hitting its physical limits and drawing excessive current.

## Try This! (Challenges)
1. **Interactive OLED Status**: Add an OLED screen (Project 60) and display a simulated dial representing the servo position.
2. **Angle sweep button**: Add a button to sweep the servo back and forth automatically between 0° and 180°.
3. **Buzzer limit alert**: Sound a quick beep on a buzzer (GPIO 15) when the servo reaches 0° or 180° limits.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move | Vin power missing | Confirm that the servo is powered from the 5V Vin rail, not the 3.3V rail |
| OLED screen stays black | I2C address mismatch | Verify the I2C address in `display.begin()` matches your OLED module (usually `0x3C`) |
| Servo jitter | Noise on signal line | Install a 100 µF capacitor across the servo power lines to filter noise |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [55 - ESP32 Web Page Buzzer frequency selector](55-esp32-web-page-buzzer-frequency-selector.md)
- [57 - ESP32 Web Page DC Motor speed dial](57-esp32-web-page-dc-motor-speed-dial.md) (Next project)
- [97 - ESP32 Dual Axis Joystick Servos](../intermediate/97-esp32-dual-axis-joystick-servos.md)
