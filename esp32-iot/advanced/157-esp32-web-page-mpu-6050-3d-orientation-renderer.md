# 157 - Web Page MPU-6050 3D Orientation renderer (Three.js demo)

Build an interactive 3D orientation visualizer on the ESP32 that reads tilt angles from an MPU-6050 IMU over I2C (GPIO 21/22), and hosts a web dashboard containing a Three.js 3D canvas rendering a solid color block that rotates in real time matching the physical sensor's pitch and roll.

## Goal
Learn how to interface MPU-6050 sensors, execute gravity vector trigonometry to calculate pitch and roll, build Three.js 3D scenes, map orientation angles, and compile REST API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MPU-6050 sensor is on I2C. The ESP32 calculates pitch and roll angles. Navigating to the ESP32's IP address serves a web page that loads the Three.js 3D rendering library. The webpage renders a 3D colored block. Client-side JavaScript polls the ESP32 for orientation angles and rotates the 3D block to mirror the MPU-6050's orientation.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MPU-6050 6-Axis IMU Sensor | `i2c_device` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, pitch and roll are simulated using random sinusoidal drift cycles if the physical MPU-6050 is not present.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MPU-6050 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the MPU-6050 sensor from the 3.3V pin. Ensure SDA and SCL are connected securely to GPIO 21 and 22.

## Code
```cpp
// Web Page MPU-6050 3D Orientation renderer (Three.js 3D Scene + I2C IMU + Web API)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

Adafruit_MPU6050 mpu;
WebServer server(80);

// Orientation variables
float pitchDeg = 0.0;
float rollDeg = 0.0;

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 50; // Read sensor every 50 ms

// Read accelerometer values and compute pitch and roll
void readOrientation() {
  sensors_event_t a, g, temp;
  
  if (mpu.getEvent(&a, &g, &temp)) {
    float ax = a.acceleration.x;
    float ay = a.acceleration.y;
    float az = a.acceleration.z;
    
    // Pitch calculation (rotation around Y axis)
    pitchDeg = atan2(-ax, sqrt(ay * ay + az * az)) * 180.0 / PI;
    
    // Roll calculation (rotation around X axis)
    rollDeg = atan2(ay, az) * 180.0 / PI;
  } else {
    // Sinusoidal drift simulation for workspace testing
    static float simTime = 0.0;
    simTime += 0.05;
    pitchDeg = sin(simTime) * 30.0;
    rollDeg = cos(simTime) * 30.0;
  }
}

// HTTP API endpoint returning JSON data
void handleGetOrientation() {
  String json = "{\"pitch\":" + String(pitchDeg, 2) + 
                 ",\"roll\":" + String(rollDeg, 2) + "}";
  server.send(200, "application/json", json);
}

// Serve root webpage containing Three.js script loading
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>3D Orientation HUD</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; overflow: hidden; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 550px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  #rendererContainer { width: 100%; height: 250px; background-color: #0f172a; border-radius: 8px; border: 1px solid #334155; margin-top: 15px; position: relative; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin-top: 20px; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 12px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 22px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n";
  // Import Three.js from Cloudflare CDN
  html += "<script src=\"https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js\"></script>\n";
  html += "</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>3D Orientation Visualizer</h1>\n";
  
  // 3D Canvas Container
  html += "  <div id=\"rendererContainer\"></div>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Pitch (Y-Axis)</div><div class=\"metric-val\" id=\"pitchVal\">0.00&deg;</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Roll (X-Axis)</div><div class=\"metric-val\" id=\"rollVal\">0.00&deg;</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">WebGL Three.js rendering scene | REST Polling: 100 ms</p>\n";
  html += "</div>\n";
  
  // Three.js and AJAX Polling Script
  html += "<script>\n";
  html += "  const container = document.getElementById('rendererContainer');\n";
  
  // 1. Setup Three.js Scene, Camera, and Renderer
  html += "  const scene = new THREE.Scene();\n";
  html += "  const camera = new THREE.PerspectiveCamera(45, container.clientWidth / container.clientHeight, 0.1, 100);\n";
  html += "  camera.position.z = 6;\n";
  
  html += "  const renderer = new THREE.WebGLRenderer({ antialias: true });\n";
  html += "  renderer.setSize(container.clientWidth, container.clientHeight);\n";
  html += "  container.appendChild(renderer.domElement);\n";
  
  // 2. Add Light sources
  html += "  const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);\n";
  html += "  scene.add(ambientLight);\n";
  html += "  const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);\n";
  html += "  dirLight.position.set(5, 5, 5);\n";
  html += "  scene.add(dirLight);\n";
  
  // 3. Create 3D Block Geometry (Robotic Board simulation shape)
  html += "  const geometry = new THREE.BoxGeometry(3, 0.2, 1.8);\n";
  // Apply individual face colors
  html += "  const materials = [\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0xef4444 }), // right\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0x3b82f6 }), // left\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0x10b981 }), // top\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0xf59e0b }), // bottom\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0x38bdf8 }), // front\n";
  html += "    new THREE.MeshLambertMaterial({ color: 0x8b5cf6 })  // back\n";
  html += "  ];\n";
  html += "  const cube = new THREE.Mesh(geometry, materials);\n";
  html += "  scene.add(cube);\n";
  
  // Render loop
  html += "  function animate() {\n";
  html += "    requestAnimationFrame(animate);\n";
  html += "    renderer.render(scene, camera);\n";
  html += "  }\n";
  html += "  animate();\n";
  
  // Adjust viewport on resize
  html += "  window.addEventListener('resize', () => {\n";
  html += "    camera.aspect = container.clientWidth / container.clientHeight;\n";
  html += "    camera.updateProjectionMatrix();\n";
  html += "    renderer.setSize(container.clientWidth, container.clientHeight);\n";
  html += "  });\n";
  
  // 4. Update rotation coordinates via AJAX GET requests
  html += "  function updateOrientation() {\n";
  html += "    fetch('/api/orientation')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('pitchVal').innerHTML = data.pitch.toFixed(2) + '&deg;';\n";
  html += "        document.getElementById('rollVal').innerHTML = data.roll.toFixed(2) + '&deg;';\n";
  
  // Convert degrees to radians for Three.js mesh rotation
  // Three.js X-axis corresponds to Roll, Z-axis corresponds to Pitch
  html += "        cube.rotation.z = (data.pitch * Math.PI) / 180.0;\n";
  html += "        cube.rotation.x = (data.roll * Math.PI) / 180.0;\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateOrientation();\n";
  html += "    setInterval(updateOrientation, 100); // Poll 10 times a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Wire.begin();
  if (!mpu.begin()) {
    Serial.println("[IMU Error] MPU-6050 sensor missing! Running in simulation mode.");
  }
  
  Serial.println("\nESP32 3D Renderer starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/orientation", handleGetOrientation);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  lastSampleTime = millis();
}

void loop() {
  server.handleClient();
  
  // Sample orientation periodically (every 50 ms)
  unsigned long now = millis();
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    lastSampleTime = now;
    readOrientation();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. In simulation, since the MPU-6050 is simulated using built-in drift functions, click **Run** to launch.
3. Open your web browser and navigate to the printed IP address.
4. Verify that a colored 3D rectangular prism renders inside the dashboard box.
5. Verify that the 3D block rotates dynamically matching the simulated pitch and roll angles printed on the webpage.

## Expected Output
Serial Monitor:
```
[IMU Error] MPU-6050 sensor missing! Running in simulation mode.
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
```

Browser JSON Console (`/api/orientation`):
```json
{"pitch":15.42,"roll":-5.83}
```

## Expected Canvas Behavior
* The simulated IMU coordinates rotate the Three.js 3D block mesh in the client browser canvas dynamically.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pitchDeg = atan2(...)` | Computes the pitch angle in degrees from raw acceleration vectors. |
| `rollDeg = atan2(...)` | Computes the roll angle in degrees from raw acceleration vectors. |
| `new THREE.WebGLRenderer(...)` | Initializes the WebGL context in the webpage container. |
| `cube.rotation.x = ...` | Maps the Roll angle (radians) to rotation around the cube's X-axis. |

## Hardware & Safety Concept: Gravity Vector Alignment and Gimbal Lock
* **Gravity Vectors**: Accelerometer orientation calculation assumes the only acceleration acting on the sensor is gravity (approx. 9.8 m/s²). If the sensor experiences centrifugal force or vibration (like when mounted on a moving drone), the computed pitch and roll angles will be corrupted. In moving systems, integrate gyroscope rates using a Complementary Filter (Project 155) to maintain accurate orientation.
* **Gimbal Lock**: Euler angles (Pitch, Roll, Yaw) suffer from gimbal lock (loss of one degree of freedom when Pitch reaches 90°). For orientation tracking through complete 3D rotations, use **quaternions** instead of Euler angles to represent 3D orientations without gimbal lock.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a 2D crosshair representing the tilt direction.
2. **Tilt alarm buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the roll angle exceeds 45 degrees.
3. **Control led brightness**: Use the pitch angle to control the brightness of an LED (GPIO 12) via PWM.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The 3D canvas is empty and black | WebGL not supported | Ensure your browser supports WebGL and hardware acceleration is enabled |
| The block rotates in the wrong direction | Axis mapping inverted | Swap the signs of the rotations in JavaScript (e.g. `cube.rotation.z = -(data.pitch * Math.PI) / 180.0`) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [155 - ESP32 Web Page Complementary Filter balancer](155-esp32-web-page-complementary-filter-balancer.md)
- [156 - ESP32 Web Page Sound Level FFT analyzer](156-esp32-web-page-sound-level-fft-analyzer.md)
- [158 - Keypad RFID Multi-factor Lock IoT with Remote Open override](158-esp32-keypad-rfid-multi-factor-lock-iot-with-remote-open-url-override.md) (Next project)
