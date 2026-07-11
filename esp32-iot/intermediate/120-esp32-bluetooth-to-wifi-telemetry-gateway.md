# 120 - Bluetooth to WiFi Telemetry Gateway

Build a communication gateway on the ESP32 that runs Bluetooth Classic (BluetoothSerial) and WiFi simultaneously. It reads local analog sensor telemetry on GPIO 34, transmits data packets over Bluetooth, processes incoming Bluetooth command strings, and hosts a WiFi web page displaying live connection states.

## Goal
Learn how to run Bluetooth Serial and WiFi network stacks simultaneously, parse custom incoming packet structures, relay data between protocols, and display status info on a web console.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and starts a Bluetooth Classic node named `ESP32_Gateway`. An analog sensor is on GPIO 34, and an LED on GPIO 12. Telemetry is sent over Bluetooth. Bluetooth terminal commands (like `LED_ON`/`LED_OFF`) control the LED. Simultaneously, the ESP32 hosts a webpage displaying the latest telemetry and the last Bluetooth command received.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (Sensor) | `potentiometer` | Yes | Yes |
| LED (Status) | `led` | Yes | Yes |
| 220 Ω Resistor | `resistor` | No | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **MbedO Note:** In simulation, Bluetooth connections are simulated using serial logging, while the WiFi web console remains fully interactive.

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | OUT (Analog) | GPIO34 | Yellow | Telemetry signal |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared Resistor | Cathode (-) | GND | Black | Current limiting |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the potentiometer from the 3.3V rail. Connect a 220 Ω resistor in series with the LED.

## Code
```cpp
// Bluetooth to WiFi Telemetry Gateway (Simultaneous BT Serial + WiFi Web Portal)
#include <WiFi.h>
#include <WebServer.h>
#include <BluetoothSerial.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int SENSOR_PIN = 34;
const int LED_PIN = 12;

BluetoothSerial SerialBT;
WebServer server(80);

int sensorValue = 0;
String lastBTCommand = "None";
bool ledState = false;

// HTTP API endpoint returning JSON data
void handleGetGatewayData() {
  String json = "{\"sensor\":" + String(sensorValue) + 
                 ",\"led\":" + String(ledState ? 1 : 0) +
                 ",\"btCmd\":\"" + lastBTCommand + "\"}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Telemetry Gateway Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .grid { display: grid; grid-template-columns: 1fr; gap: 15px; margin: 20px 0; }\n";
  html += "  .metric-card { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; }\n";
  html += "  .metric-label { font-size: 11px; color: #64748b; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .metric-val { font-size: 24px; font-weight: bold; margin-top: 5px; color: #10b981; font-family: monospace; }\n";
  html += "  .status { display: flex; justify-content: space-between; align-items: center; margin-top: 5px; }\n";
  html += "  .badge { padding: 4px 8px; border-radius: 12px; font-size: 11px; font-weight: bold; text-transform: uppercase; }\n";
  html += "  .active { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .inactive { background-color: #991b1b; color: #fee2e2; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Telemetry Gateway</h1>\n";
  
  html += "  <div class=\"grid\">\n";
  html += "    <div class=\"metric-card\">\n";
  html += "      <div class=\"metric-label\">Analog Telemetry</div>\n";
  html += "      <div class=\"metric-val\" id=\"sensorVal\">0</div>\n";
  html += "    </div>\n";
  html += "    <div class=\"metric-card\">\n";
  html += "      <div class=\"metric-label\">Last Bluetooth Command</div>\n";
  html += "      <div class=\"metric-val\" id=\"btVal\" style=\"color:#38bdf8;\">None</div>\n";
  html += "    </div>\n";
  html += "    <div class=\"metric-card\">\n";
  html += "      <div class=\"metric-label\">Local Actuator State</div>\n";
  html += "      <div class=\"status\">\n";
  html += "        <span style=\"font-weight:bold;\">LED Light Pin</span>\n";
  html += "        <span id=\"ledBadge\" class=\"badge inactive\">OFF</span>\n";
  html += "      </div>\n";
  html += "    </div>\n";
  html += "  </div>\n";
  
  html += "  <p class=\"footer\">BT node: ESP32_Gateway | WiFi Web Portal active</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateGateway() {\n";
  html += "    fetch('/api/data')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        document.getElementById('sensorVal').innerText = data.sensor;\n";
  html += "        document.getElementById('btVal').innerText = data.btCmd;\n";
  
  html += "        const bdgEl = document.getElementById('ledBadge');\n";
  html += "        if (data.led === 1) {\n";
  html += "          bdgEl.innerText = 'ON';\n";
  html += "          bdgEl.className = 'badge active';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'OFF';\n";
  html += "          bdgEl.className = 'badge inactive';\n";
  html += "        }\n";
  
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateGateway();\n";
  html += "    setInterval(updateGateway, 1000); // Poll once every second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(SENSOR_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with LED OFF
  
  // 1. Initialize Bluetooth Classic Serial interface
  if (!SerialBT.begin("ESP32_Gateway")) {
    Serial.println("[BT Error] Failed to initialize Bluetooth Classic!");
  } else {
    Serial.println("[Bluetooth] Initialized. Device name: ESP32_Gateway");
  }
  
  Serial.println("\nESP32 Telemetry Gateway Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/data", handleGetGatewayData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  
  // 2. Sample analog sensor level
  sensorValue = analogRead(SENSOR_PIN);
  
  // 3. Read incoming command packets from Bluetooth link
  if (SerialBT.available()) {
    String cmd = SerialBT.readStringUntil('\n');
    cmd.trim();
    
    if (cmd.length() > 0) {
      lastBTCommand = cmd;
      Serial.printf("[Bluetooth] Command received: %s\n", cmd.c_str());
      
      // Control LED based on command
      if (cmd == "LED_ON") {
        ledState = true;
        digitalWrite(LED_PIN, HIGH);
      } 
      else if (cmd == "LED_OFF") {
        ledState = false;
        digitalWrite(LED_PIN, LOW);
      }
    }
  }
  
  // 4. Send telemetry data over Bluetooth periodically
  static unsigned long lastBtTransmit = 0;
  if (millis() - lastBtTransmit >= 1000) {
    // Send standard telemetry packet format
    SerialBT.printf("SENS:%d\n", sensorValue);
    lastBtTransmit = millis();
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer** (to simulate the sensor), and **LED** onto the canvas.
2. Wire Potentiometer to **GPIO34** and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Adjust the potentiometer widget.
5. Verify that the telemetry readings update on the web page.
6. Open your terminal or a simulated Bluetooth client. Send the string `LED_ON` to the Bluetooth Serial. Verify that the LED on the canvas turns ON and the webpage displays `LED_ON` as the last command.

## Expected Output
Serial Monitor:
```
[Bluetooth] Initialized. Device name: ESP32_Gateway
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Bluetooth] Command received: LED_ON
```

## Expected Canvas Behavior
* Adjusting the potentiometer updates the live telemetry reading on the web page.
* Sending command strings over simulated Bluetooth toggles the LED widget.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SerialBT.begin("ESP32_Gateway")` | Initializes Bluetooth Classic with the broadcast device name. |
| `SerialBT.available()` | Checks if data bytes have arrived over the Bluetooth serial channel. |
| `SerialBT.readStringUntil('\n')` | Reads incoming characters until a newline character is received. |
| `SerialBT.printf("SENS:%d\n", ...)` | Transmits telemetry data over the Bluetooth serial channel. |

## Hardware & Safety Concept: Coexistence and Antenna Multiplexing
* **Antenna Multiplexing**: The ESP32 features a single RF radio block and antenna shared between WiFi and Bluetooth. Running both protocols simultaneously (called coexistence) can cause packet collisions or slow speeds.
* **Coexistence Management**: If you need high-bandwidth WiFi streaming alongside high-speed Bluetooth audio, use an external Bluetooth module (like the HC-05) connected to Serial2, letting the ESP32 handle WiFi on its internal radio.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display both WiFi and Bluetooth status icons.
2. **Alert Buzzer**: Add a buzzer (GPIO 15) and sound an alarm if the sensor value exceeds 3000.
3. **SPIFFS integration**: Log received Bluetooth commands to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth does not pair | Name mismatch | Confirm that your smartphone's Bluetooth scan lists the device name `ESP32_Gateway` |
| WiFi drops out during Bluetooth transmission | RF radio conflict | Ensure that your router uses a different channel or update the coexistence configuration |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [48 - ESP32 LDR light meter remote report](../beginner/48-esp32-ldr-light-meter-remote-report.md)
- [118 - ESP32 IR Remote Web LED Control](118-esp32-ir-remote-web-led-control.md)
- [121 - ESP32 WiFi Web Page Drive Controller](121-esp32-wifi-web-page-drive-controller.md) (Next project)
