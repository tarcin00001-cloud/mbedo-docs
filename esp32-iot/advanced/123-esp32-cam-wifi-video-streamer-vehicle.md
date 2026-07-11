# 123 - WiFi Video Streamer Vehicle (ESP32-CAM interface integration)

Build a web-controlled surveillance robot on the ESP32-CAM module that captures real-time video frames using an OV2640 camera, hosts a low-latency MJPEG video stream alongside a motor control interface, and drives two DC motors via an L298N driver on GPIOs 12, 13, 14, and 15.

## Goal
Learn how to configure the ESP32-CAM module, initialize the OV2640 camera, handle HTTP multipart stream responses (`multipart/x-mixed-replace`), and write motor control logic using the module's available I/O pins.

## What You Will Build
An ESP32-CAM module connects to a local WiFi network. An L298N driver on GPIOs 12, 13, 14, and 15 controls two DC motors. The ESP32-CAM hosts a webpage on port 80 displaying a live video stream from the camera. The webpage features control buttons to drive the robot. Pressing buttons controls the motors while displaying the camera's feed.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32-CAM Module | `esp32` | Yes | Yes |
| OV2640 Camera Module | `camera` | Yes | Yes |
| L298N H-Bridge Motor Driver | `l298n` | Yes | Yes |
| DC Gear Motors | `motor_dc` | Yes (2 pieces) | Yes (2 pieces) |
| Shared power / Battery Pack | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, a placeholder image stream is fed to the webpage, while the motor widgets actuate based on input commands.

## Wiring

| Component | Component Pin | ESP32-CAM Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| L298N Driver | IN1 / IN2 | GPIO12 / GPIO13 | Yellow / Green | Left motor control |
| L298N Driver | IN3 / IN4 | GPIO14 / GPIO15 | Blue / White | Right motor control |
| L298N Driver | GND | GND | Black | Common ground rail |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The ESP32-CAM board has limited available GPIO pins because many pins are shared with the camera connector and SD card slot. Use GPIOs 12, 13, 14, and 15 for motor control.

## Code
```cpp
// WiFi Video Streamer Vehicle (ESP32-CAM Video Stream + Motor Control Console)
#include <WiFi.h>
#include <WebServer.h>
#include "esp_camera.h"

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// L298N motor driver pins on ESP32-CAM board
const int IN1 = 12;
const int IN2 = 13;
const int IN3 = 14;
const int IN4 = 15;

WebServer server(80);

// Camera Model configuration pins (AI-Thinker Board Model)
#define PWDN_GPIO_NUM     32
#define RESET_GPIO_NUM    -1
#define XCLK_GPIO_NUM      0
#define SIOD_GPIO_NUM     26
#define SIOC_GPIO_NUM     27
#define Y9_GPIO_NUM       35
#define Y8_GPIO_NUM       34
#define Y7_GPIO_NUM       39
#define Y6_GPIO_NUM       36
#define Y5_GPIO_NUM       21
#define Y4_GPIO_NUM       19
#define Y3_GPIO_NUM       18
#define Y2_GPIO_NUM        5
#define VSYNC_GPIO_NUM    25
#define HREF_GPIO_NUM     23
#define PCLK_GPIO_NUM     22

// Movement control routines
void moveForward() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void moveBackward() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void turnLeft() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, HIGH);
  digitalWrite(IN3, HIGH);
  digitalWrite(IN4, LOW);
}

void turnRight() {
  digitalWrite(IN1, HIGH);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, HIGH);
}

void stopRobot() {
  digitalWrite(IN1, LOW);
  digitalWrite(IN2, LOW);
  digitalWrite(IN3, LOW);
  digitalWrite(IN4, LOW);
}

// MJPEG Stream HTTP Response Boundary Header
#define PART_BOUNDARY "123456789000000000000987654321"
static const char* _STREAM_CONTENT_TYPE = "multipart/x-mixed-replace;boundary=" PART_BOUNDARY;
static const char* _STREAM_BOUNDARY = "\r\n--" PART_BOUNDARY "\r\n";
static const char* _STREAM_PART = "Content-Type: image/jpeg\r\nContent-Length: %u\r\n\r\n";

// HTTP Handler to output live MJPEG video frames stream
void handleVideoStream() {
  WiFiClient client = server.client();
  
  // Send HTTP Header response
  client.print("HTTP/1.1 200 OK\r\n");
  client.print("Content-Type: ");
  client.print(_STREAM_CONTENT_TYPE);
  client.print("\r\n\r\n");
  
  Serial.println("[Stream] Client connected. Streaming video frames...");
  
  while (client.connected()) {
    camera_fb_t* fb = esp_camera_fb_get();
    if (!fb) {
      Serial.println("[Camera] Failed to capture frame!");
      delay(100);
      continue;
    }
    
    // Send boundary marker
    client.print(_STREAM_BOUNDARY);
    
    // Send frame header
    char headerBuffer[64];
    sprintf(headerBuffer, _STREAM_PART, fb->len);
    client.print(headerBuffer);
    
    // Write frame bytes buffer
    client.write(fb->buf, fb->len);
    
    // Return frame buffer to camera memory
    esp_camera_fb_return(fb);
    
    // Limit frame rate (approx 15 FPS)
    delay(60);
  }
  
  Serial.println("[Stream] Client disconnected.");
}

// HTTP POST endpoint to update movement state
void handlePostAction() {
  if (server.hasArg("cmd")) {
    String cmd = server.arg("cmd");
    if (cmd == "forward")  moveForward();
    else if (cmd == "reverse") moveBackward();
    else if (cmd == "left")    turnLeft();
    else if (cmd == "right")   turnRight();
    else                       stopRobot();
    
    server.send(200, "text/plain", "OK");
  } else {
    server.send(400, "text/plain", "Bad Request");
  }
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Surveillance Robot Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  
  // Styled stream frame box
  html += "  .video-container { width: 100%; aspect-ratio: 4/3; background-color: #0f172a; border: 2px solid #334155; border-radius: 12px; overflow: hidden; margin-bottom: 25px; display: flex; justify-content: center; align-items: center; }\n";
  html += "  .video-feed { width: 100%; height: 100%; object-fit: cover; }\n";
  
  html += "  .control-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 10px; margin: 0 auto; max-width: 260px; }\n";
  html += "  .btn { padding: 16px; font-size: 18px; font-weight: bold; color: white; background-color: #334155; border: 1px solid #475569; border-radius: 12px; cursor: pointer; transition: background-color 0.1s, transform 0.1s; }\n";
  html += "  .btn:active { background-color: #38bdf8; color: #0f172a; transform: scale(0.95); }\n";
  html += "  .btn-stop { background-color: #991b1b; border-color: #b91c1c; }\n";
  html += "  .empty { visibility: hidden; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Surveillance Robot Console</h1>\n";
  
  // Live Feed
  html += "  <div class=\"video-container\">\n";
  html += "    <img class=\"video-feed\" src=\"/stream\" alt=\"Live Video Feed\">\n";
  html += "  </div>\n";
  
  // D-Pad Grid
  html += "  <div class=\"control-grid\">\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('forward')\" onmouseup=\"sendCmd('stop')\">&#9650;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('left')\" onmouseup=\"sendCmd('stop')\">&#9664;</button>\n";
  html += "    <button class=\"btn btn-stop\" onclick=\"sendCmd('stop')\">STOP</button>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('right')\" onmouseup=\"sendCmd('stop')\">&#9654;</button>\n";
  
  html += "    <div class=\"empty\"></div>\n";
  html += "    <button class=\"btn\" onmousedown=\"sendCmd('reverse')\" onmouseup=\"sendCmd('stop')\">&#9660;</button>\n";
  html += "    <div class=\"empty\"></div>\n";
  html += "  </div>\n";
  
  html += "</div>\n";
  
  // Script
  html += "<script>\n";
  html += "  function sendCmd(cmdVal) {\n";
  html += "    const body = new URLSearchParams();\n";
  html += "    body.append('cmd', cmdVal);\n";
  html += "    fetch('/action', { method: 'POST', body: body });\n";
  html += "  }\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Configure motor driver pins
  pinMode(IN1, OUTPUT);
  pinMode(IN2, OUTPUT);
  pinMode(IN3, OUTPUT);
  pinMode(IN4, OUTPUT);
  stopRobot();
  
  // 1. Configure camera initialization parameters
  camera_config_t config;
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk = XCLK_GPIO_NUM;
  config.pin_pclk = PCLK_GPIO_NUM;
  config.pin_vsync = VSYNC_GPIO_NUM;
  config.pin_href = HREF_GPIO_NUM;
  config.pin_sscb_sda = SIOD_GPIO_NUM;
  config.pin_sscb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn = PWDN_GPIO_NUM;
  config.pin_reset = RESET_GPIO_NUM;
  config.xclk_freq_hz = 20000000;
  config.pixel_format = PIXFORMAT_JPEG;
  
  // Frame frame parameters
  config.frame_size = FRAMESIZE_QVGA; // 320x240 frame resolution
  config.jpeg_quality = 12;          // 0-63 (lower is higher quality)
  config.fb_count = 2;               // Use double buffering
  
  // 2. Initialize camera module
  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("[Camera Error] Failed to initialize camera! Code: 0x%x\n", err);
    // Don't halt, let server run
  } else {
    Serial.println("[Camera] Initialized successfully.");
  }
  
  Serial.println("\nESP32-CAM Streaming Robot starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/stream", handleVideoStream);
  server.on("/action", HTTP_POST, handlePostAction);
  
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
1. Drag **ESP32-CAM** (or ESP32 DevKitC configured with camera settings), **L298N Driver**, and **2 DC Motors** onto the canvas.
2. Wire IN1–IN4 to **GPIO12, 13, 14, and 15** respectively.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Verify that the video stream placeholder displays on the webpage.
6. Click and hold the D-Pad buttons. Verify that the simulated motors spin in the correct directions.

## Expected Output
Serial Monitor:
```
[Camera] Initialized successfully.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Stream] Client connected. Streaming video frames...
```

## Expected Canvas Behavior
* Click-and-hold actions spin the motor widgets on the canvas in matching directions.
* Releasing buttons stops the motors.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `esp_camera_init(&config)` | Initializes the OV2640 camera registers. |
| `esp_camera_fb_get()` | Captures a JPEG frame from the camera. |
| `client.print(_STREAM_BOUNDARY)` | Prints the HTTP boundary tag separating video frames. |
| `esp_camera_fb_return(fb)` | Frees the frame buffer back to the camera memory pool. |

## Hardware & Safety Concept: Bandwidth Optimization and Pin Constraints
* **Bandwidth Optimization**: Streaming video is resource-intensive. Using QVGA (320x240) resolution at 15 FPS consumes around 1.5 Mbps of bandwidth, keeping the stream smooth on typical local networks. Avoid higher resolutions (like UXGA) on mobile robots to prevent severe lag.
* **Pin Constraints**: ESP32-CAM boards have very few available pins because the camera uses 15 GPIOs. Do not use GPIO 4 (the high-brightness flash LED) for other functions, as doing so will turn on the flash whenever you use that pin.

## Try This! (Challenges)
1. **Flash Control**: Add a button on the webpage to turn the onboard flash LED (GPIO 4) ON or OFF.
2. **Speed Controls**: Implement speed selections (Low/High) using software PWM on the motor pins.
3. **Low Battery Alert**: Read the ESP32-CAM's supply voltage and display a low-power warning on the webpage.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Camera init fails (Code 0x20001) | Power issue or loose cable | Verify that the camera ribbon cable is firmly seated in the connector and the board has a clean 5V supply |
| Video stream is choppy or drops | WiFi signal weak | Reduce the resolution to `FRAMESIZE_QQVGA` or decrease JPEG quality to speed up frame times |
| ESP32-CAM gets hot | High current draw | This is normal during continuous camera streaming. Add a small heatsink to the ESP32 chip if running for long periods |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md)
- [122 - ESP32 WebSocket low-latency Drive Controller](122-esp32-websocket-low-latency-drive-controller.md)
- [124 - ESP32 Obstacle Avoidance Robot with Web HUD](124-esp32-obstacle-avoidance-robot-with-web-hud-telemetry.md) (Next project)
