# 57 - Web Page DC Motor Speed Dial

Build a web-controlled motor speed station on the ESP32 that hosts a control panel, parses target speed percentage parameters from HTTP requests (`/set?speed=80`), and drives a DC motor via an L298N motor driver using the hardware-timed LEDC PWM peripheral.

## Goal
Learn how to parse query parameters, generate PWM signals using the ESP32 hardware LEDC API, drive DC motor drivers (such as the L298N), and handle redirects.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An L298N motor driver and DC motor are connected to GPIO 13 (Speed ENA), GPIO 12 (IN1), and GPIO 14 (IN2). Navigating to the ESP32's IP address displays a webpage with a slider (0% to 100%). Submitting a speed maps the value to the PWM duty cycle, adjusting the motor speed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| L298N Motor Driver Module | `l298n` | Yes | Yes |
| DC Motor | `motor` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | ENA (Speed) | GPIO13 | Orange | PWM speed signal |
| L298N Driver | IN1 (Direction) | GPIO12 | Yellow | Direction control pin 1 |
| L298N Driver | IN2 (Direction) | GPIO14 | Green | Direction control pin 2 |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the L298N driver from the 5V Vin rail.

## Code
```cpp
// Web Page DC Motor speed dial (LEDC PWM Motor Speed Controller)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Configure Motor Pins
const int ENA_PIN = 13; // Speed control
const int IN1_PIN = 12; // Direction pin 1
const int IN2_PIN = 14; // Direction pin 2

// LEDC PWM parameters
const int pwmChannel = 0;
const int pwmFreq = 5000;
const int pwmResolution = 8; // 8-bit resolution (0 - 255)

// Active motor speed percentage state variable
int activeSpeedPercent = 0; // Default off

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  // Compile HTML page with speed dial controls
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>DC Motor Speed Dial</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 400px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; margin-bottom: 20px; }\n";
  html += "  .status { font-size: 18px; margin: 20px 0; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; color: " + String(activeSpeedPercent > 0 ? "#10b981" : "#ef4444") + "; }\n";
  html += "  .slider { width: 100%; margin: 20px 0; }\n";
  html += "  .btn { display: inline-block; padding: 12px 24px; font-size: 15px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn:hover { background-color: #0ea5e9; }\n";
  html += "  .presets { display: flex; justify-content: space-between; margin-top: 20px; border-top: 1px solid #334155; padding-top: 20px; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>DC Motor Speed Dial</h1>\n";
  
  // Status display
  html += "  <p class=\"status\">Current Speed: <span class=\"status-val\">" + String(activeSpeedPercent) + "%</span></p>\n";
  
  // Slider form for custom speed selection
  html += "  <form action=\"/set\" method=\"GET\">\n";
  html += "    <input type=\"range\" id=\"speedRange\" name=\"speed\" min=\"0\" max=\"100\" value=\"" + String(activeSpeedPercent) + "\" class=\"slider\" oninput=\"updateSliderVal(this.value)\">\n";
  html += "    <span id=\"sliderVal\">" + String(activeSpeedPercent) + "%</span><br><br>\n";
  html += "    <button type=\"submit\" class=\"btn\">Set Speed</button>\n";
  html += "  </form>\n";
  
  // Presets
  html += "  <div class=\"presets\">\n";
  html += "    <a href=\"/set?speed=0\" class=\"btn\" style=\"background-color: #ef4444;\">STOP</a>\n";
  html += "    <a href=\"/set?speed=50\" class=\"btn\">50%</a>\n";
  html += "    <a href=\"/set?speed=100\" class=\"btn\">100%</a>\n";
  html += "  </div>\n";
  
  html += "</div>\n";
  
  // JS script to update text value dynamically as slider moves
  html += "<script>\n";
  html += "  function updateSliderVal(val) {\n";
  html += "    document.getElementById('sliderVal').innerText = val + '%';\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Speed setting handler ("/set")
void handleSet() {
  if (server.hasArg("speed")) {
    int speedPercent = server.arg("speed").toInt();
    
    // Validate target speed percent within bounds
    if (speedPercent >= 0 && speedPercent <= 100) {
      activeSpeedPercent = speedPercent;
      
      // Map percentage (0 - 100) to 8-bit PWM duty cycle (0 - 255)
      int dutyCycle = map(activeSpeedPercent, 0, 100, 0, 255);
      
      // Write duty cycle to LEDC channel
      ledcWrite(pwmChannel, dutyCycle);
      
      // Enforce forward rotation direction on driver pins
      if (activeSpeedPercent > 0) {
        digitalWrite(IN1_PIN, HIGH);
        digitalWrite(IN2_PIN, LOW);
      } else {
        digitalWrite(IN1_PIN, LOW);
        digitalWrite(IN2_PIN, LOW);
      }
      
      Serial.printf("[Server] Motor speed updated: %d%% (Duty: %d)\n", activeSpeedPercent, dutyCycle);
    } else {
      Serial.printf("[Server Error] Out of bounds speed attempted: %d%%\n", speedPercent);
    }
  }
  
  // Redirect back to root page to update status
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(IN1_PIN, OUTPUT);
  pinMode(IN2_PIN, OUTPUT);
  
  digitalWrite(IN1_PIN, LOW);
  digitalWrite(IN2_PIN, LOW);
  
  // Configure LEDC PWM channel and attach ENA pin
  ledcSetup(pwmChannel, pwmFreq, pwmResolution);
  ledcAttachPin(ENA_PIN, pwmChannel);
  
  // Set initial speed to 0 (Stopped)
  ledcWrite(pwmChannel, 0);
  
  Serial.println("\nESP32 Motor Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
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
1. Drag **ESP32 DevKitC**, **L298N Motor Driver**, and **DC Motor** onto the canvas.
2. Wire ENA to **GPIO13**, IN1 to **GPIO12**, and IN2 to **GPIO14**. Wire the motor outputs of L298N to the DC Motor.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the slider or click the preset buttons to control the motor speed.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Motor speed updated: 50% (Duty: 127)
[Server] Motor speed updated: 100% (Duty: 255)
[Server] Motor speed updated: 0% (Duty: 0)
```

## Expected Canvas Behavior
* Adjusting the slider on the web page updates the speed and rotation of the DC Motor widget.
* Clicking STOP stops the motor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `ledcSetup(...)` | Initializes the LEDC hardware timer channel with the target frequency and resolution. |
| `ledcAttachPin(...)` | Binds the target GPIO pin to the configured LEDC channel. |
| `map(speed, 0, 100, 0, 255)` | Converts the 0% to 100% range to the 8-bit duty cycle scale (0 to 255). |
| `ledcWrite(pwmChannel, ...)` | Configures the output duty cycle on the active channel. |

## Hardware & Safety Concept: Hardware-Timed PWM vs Software PWM
Driving motors requires a high-frequency PWM signal to prevent acoustic noise and motor hum. Using software timers to toggle pins can cause signal distortion if the CPU is busy with WiFi tasks. The ESP32's **LEDC** peripheral generates the PWM signal in hardware, running independently of the CPU to ensure stable operation.

## Try This! (Challenges)
1. **OLED Speed Display**: Add an OLED screen (Project 60) and display the current speed.
2. **Direction Control Switch**: Add a button to reverse the motor rotation direction.
3. **Buzzer confirmation tone**: Sound a warning beep (GPIO 15) if the motor speed exceeds 90%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor hums but does not rotate | PWM frequency too high or duty cycle too low | Increase the minimum speed setting or reduce the PWM frequency |
| Motor does not rotate | Driver power missing | Confirm that the motor driver is powered from the 5V Vin rail |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [56 - ESP32 Web Page Servo angle controller](56-esp32-web-page-servo-angle-controller-oled-display-feedback.md)
- [88 - ESP32 L298N Motor Driver Control](../../basic/intermediate/88-esp32-l298n-motor-driver-control.md)
- [188 - ESP32 MCPWM Motor Control](../../basic/expert/188-esp32-mcpwm-motor-control.md)
