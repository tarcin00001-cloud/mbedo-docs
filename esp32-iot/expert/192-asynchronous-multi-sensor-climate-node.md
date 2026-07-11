# 192 - Asynchronous Multi-Sensor Climate Node (asyncio reads DHT22 + Web uploads)

Build a non-blocking multi-sensor climate telemetry station on the ESP32 using cooperative multitasking loops to execute asynchronous sensor sampling, status LED blink rates, and cloud database updates concurrently without delaying the local web server.

## Goal
Learn how to implement cooperative multitasking schedulers, execute non-blocking timed tasks, manage multiple sensor acquisition loops, and host high-concurrency local web nodes.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It acts as a multi-sensor station reading a DHT22 sensor (GPIO 14), an MQ-2 smoke sensor (GPIO 34), and an LDR light sensor (GPIO 35). A cooperative scheduler runs on a single core, managing tasks at different frequencies without using the blocking `delay()` function. The ESP32 hosts a responsive web dashboard on port 80 that remains fully responsive even during background cloud uploads.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| MQ-2 Gas/Smoke Sensor | `mq2` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| MQ-2 Sensor | AO (Analog Out) | GPIO34 | Green | Smoke sensor analog input |
| LDR | AO (Analog Out) | GPIO35 | Blue | Light sensor analog input |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 10 kΩ pull-down resistor in series with the LDR to form a voltage divider circuit.

## Code
```cpp
// Asynchronous Multi-Sensor Climate Node (Cooperative multitasking scheduler + non-blocking timers)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int MQ2_PIN = 34;
const int LDR_PIN = 35;
const int STATUS_LED = 12;

WebServer server(80);

// Global sensor values
float tempC = 0.0;
float humidPct = 0.0;
float gasPct = 0.0;
float lightPct = 0.0;

// Task scheduler structure for cooperative multitasking
struct AsyncTask {
  unsigned long intervalMs;
  unsigned long lastRunTime;
  void (*taskFunction)();
};

// Asynchronous Task Functions
void taskReadDHT22();
void taskReadAnalogSensors();
void taskCloudUpload();
void taskBlinkLED();

// Initialize Task Scheduler Table
AsyncTask tasks[] = {
  { 2000, 0, taskReadDHT22 },          // Read DHT22 every 2 seconds
  { 500,  0, taskReadAnalogSensors },  // Read MQ-2 and LDR every 500 ms
  { 10000, 0, taskCloudUpload },        // Upload to cloud every 10 seconds
  { 1000, 0, taskBlinkLED }            // Blink LED at 1 Hz (50% duty cycle)
};
const int NUM_TASKS = sizeof(tasks) / sizeof(AsyncTask);

// Task 1: Read DHT22 Climate Sensor
void taskReadDHT22() {
  float t = dht.readTemperature();
  float h = dht.readHumidity();
  
  if (!isnan(t) && !isnan(h)) {
    tempC = t;
    humidPct = h;
    Serial.printf("[Task DHT22] Temp: %.1f C | Humidity: %.1f%%\n", tempC, humidPct);
  }
}

// Task 2: Read Analog Gas and Light Sensors
void taskReadAnalogSensors() {
  int rawGas = analogRead(MQ2_PIN);
  gasPct = (rawGas / 4095.0) * 100.0;
  
  int rawLight = analogRead(LDR_PIN);
  lightPct = (rawLight / 4095.0) * 100.0;
  
  Serial.printf("[Task Analog] Gas: %.1f%% | Light: %.1f%%\n", gasPct, lightPct);
}

// Task 3: Upload Climate logs to remote Cloud API
void taskCloudUpload() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    String jsonPayload = "{\"temp\":" + String(tempC, 2) + 
                         ",\"hum\":" + String(humidPct, 1) + 
                         ",\"smoke\":" + String(gasPct, 1) + 
                         ",\"light\":" + String(lightPct, 1) + "}";
                         
    Serial.printf("[Task Cloud] Uploading payload: %s\n", jsonPayload.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200 || httpResponseCode == 201) {
      Serial.println("[Task Cloud] Upload successful.");
    } else {
      Serial.printf("[Task Cloud Error] Upload failed. Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

// Task 4: Flash Status LED (Non-blocking)
void taskBlinkLED() {
  static bool ledState = false;
  ledState = !ledState;
  digitalWrite(STATUS_LED, ledState ? HIGH : LOW);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Async Climate Hub</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 500px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; margin: 25px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .footer { text-align: center; margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Asynchronous Climate Hub</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Temperature</div><div class=\"metric-val\" id=\"tempVal\">0.0 C</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Humidity</div><div class=\"metric-val\" id=\"humVal\">0.0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Gas Level</div><div class=\"metric-val\" id=\"gasVal\">0.0%</div></div>\n";
  html += "    <div class=\"metric-card\"><div class=\"metric-label\">Light Level</div><div class=\"metric-val\" id=\"lightVal\">0.0%</div></div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">WiFi Active | Tasks Scheduler: Co-routine Loops</p>\n";
  html += "</div>\n";
  
  // AJAX data fetcher
  html += "<script>\n";
  html += "  function updateDashboard() {\n";
  html += "    fetch('/data')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('tempVal').innerText = data.temp.toFixed(1) + ' C';\n";
  html += "        document.getElementById('humVal').innerText = data.hum.toFixed(1) + '%';\n";
  html += "        document.getElementById('gasVal').innerText = data.gas.toFixed(1) + '%';\n";
  html += "        document.getElementById('lightVal').innerText = data.light.toFixed(1) + '%';\n";
  html += "      });\n";
  html += "  }\n";
  html += "  window.onload = function() {\n";
  html += "    updateDashboard();\n";
  html += "    setInterval(updateDashboard, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST endpoint serving live sensor readings in JSON format
void handleGetData() {
  String json = "{\"temp\":" + String(tempC, 2) + 
                 ",\"hum\":" + String(humidPct, 1) + 
                 ",\"gas\":" + String(gasPct, 1) + 
                 ",\"light\":" + String(lightPct, 1) + "}";
  server.send(200, "application/json", json);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\nESP32 Async Climate Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // Web Server Routes
  server.on("/", HTTP_GET, handleRoot);
  server.on("/data", HTTP_GET, handleGetData);
  server.begin();
  Serial.println("HTTP Server active.");
}

void loop() {
  // 1. Process client requests instantly
  server.handleClient();
  
  // 2. Cooperative Task Scheduler Loop
  unsigned long now = millis();
  for (int i = 0; i < NUM_TASKS; i++) {
    if (now - tasks[i].lastRunTime >= tasks[i].intervalMs) {
      tasks[i].lastRunTime = now;
      tasks[i].taskFunction(); // Execute task function pointer
    }
  }
  
  delay(1); // Brief delay yields control to internal system scheduler
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **MQ-2**, **LDR**, and **LED** onto the canvas.
2. Wire the DHT22 output to **GPIO14**, MQ-2 to **GPIO34**, LDR to **GPIO35**, and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated climate variables (temperature to 24 °C, gas level to 15%, light value to 70%).
5. Verify that the console logs sensor readings at different intervals.
6. Verify that the LED flashes exactly once per second, and the background upload runs every 10 seconds.
7. Open your web browser, navigate to the IP address, and verify that the dashboard updates dynamically.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active.
[Task Analog] Gas: 15.0% | Light: 70.0%
[Task DHT22] Temp: 24.0 C | Humidity: 55.0%
[Task Analog] Gas: 15.0% | Light: 70.0%
[Task Cloud] Uploading payload: {"temp":24.00,"hum":55.0,"smoke":15.0,"light":70.0}
[Task Cloud] Upload successful.
```

## Expected Canvas Behavior
* Running the scheduler activates the LED blinker widget. Toggling simulated slider inputs updates the web indicators and serial log outputs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `void (*taskFunction)()` | Declares a C++ function pointer type to store task callbacks. |
| `tasks[i].taskFunction()` | Dynamically calls the pointer address to execute the task. |
| `now - tasks[i].lastRunTime` | Evaluates if the interval has elapsed since the task's last run time. |

## Hardware & Safety Concept: Watchdog Resets and Non-blocking Sockets
* **Watchdog Resets**: The ESP32 runs a hardware Watchdog Timer (WDT) on both cores. If the code enters a long, blocking loop (e.g. waiting in a while loop for a sensor response or executing `delay(10000)`), the WDT will reset the CPU to recover. By using cooperative multitasking, the CPU yields control to the IDLE task, preventing WDT triggers.
* **Non-blocking Sockets**: Web requests and cloud uploads can occasionally hang if the server is offline. In production, configure short timeouts on all HTTP requests (`http.setTimeout(2000)`) to prevent a failed network call from freezing the entire cooperative scheduler.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Async Tasks: OK".
2. **Dynamic Task Tuning**: Add a route to adjust the cloud upload frequency dynamically using a slider.
3. **Task Scheduler Library**: Modify the code to use the popular `TaskScheduler` library instead of our custom scheduler array.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Webpage loads slowly | Task taking too long | If a task executes blocking calls (such as deep loops or DNS queries), it delays the main scheduler loop. Ensure all tasks run in under 50 ms |
| Sensor values do not update | Analog pin noise | Ensure the analog sensors are connected to ADC1 pins (GPIO 32–39), as ADC2 pins are disabled when WiFi is active |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [139 - IoT Weather Station Node](../advanced/139-iot-weather-station-node.md)
- [191 - Autonomous Robot HUD IoT Node](191-autonomous-robot-hud-iot-node.md)
- [193 - RFID Vault with Acceleration Safeguards & IoT Event logging](193-rfid-vault-with-acceleration-safeguards-iot-event-logging.md) (Next project)
