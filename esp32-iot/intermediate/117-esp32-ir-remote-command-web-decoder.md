# 117 - IR Remote Command Web Decoder

Build an infrared remote control decoder on the ESP32 that captures IR codes using a TSOP38238 receiver on GPIO 14, decodes key codes and protocols (such as NEC), and hosts a web dashboard with real-time logs of received button presses.

## Goal
Learn how to interface infrared receivers using `IRremote`, decode remote control signals, parse hex commands and protocols, serve web dashboards, and compile JSON API endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An IR receiver is on GPIO 14. Pressing buttons on a handheld infrared remote control emits infrared light pulses. The IR receiver decodes these pulses into hex commands. The ESP32 logs these commands and hosts a webpage displaying a history table of the decoded button events in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| IR Receiver Module (TSOP38238) | `ir_receiver` | Yes | Yes |
| Handheld IR Remote | `ir_remote` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| IR Receiver | OUT (Signal) | GPIO14 | Yellow | IR pulse decoder input |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Power the IR receiver from the 3.3V pin of the ESP32 to match its input signal voltage levels.

## Code
```cpp
// IR Remote Command Web Decoder (IR command capture + Protocol parser + Web logs)
#define DECODE_NEC // Enable NEC decoding protocol
#include <WiFi.h>
#include <WebServer.h>
#include <IRremote.hpp>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int IR_RECEIVE_PIN = 14;

WebServer server(80);

// Rolling Logs buffer (Stores last 5 entries)
String irLogs[5] = {"", "", "", "", ""};
int logCount = 0;

void addIRLog(String message) {
  for (int i = 4; i > 0; i--) {
    irLogs[i] = irLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  irLogs[0] = "[" + String(timestampS) + "s] " + message;
  
  if (logCount < 5) logCount++;
  Serial.println("[IR DECODER] " + irLogs[0]);
}

// Convert protocol code to printable label string
String getProtocolLabel(int protocolCode) {
  switch (protocolCode) {
    case NEC:     return "NEC";
    case SONY:    return "SONY";
    case RC5:     return "RC5";
    case RC6:     return "RC6";
    case PANASONIC: return "PANASONIC";
    default:      return "UNKNOWN";
  }
}

// HTTP API endpoint returning JSON data
void handleGetIRData() {
  String json = "{\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + irLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>IR Decoder Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>IR Remote Decoder</h1>\n";
  
  html += "  <h3>Received Commands Log</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Activity Logs</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Asynchronous updates polling every 500 ms | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateIRLogs() {\n";
  html += "    fetch('/api/ir')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const tableBody = document.getElementById('logTable');\n";
  html += "        tableBody.innerHTML = '';\n";
  html += "        data.logs.forEach(log => {\n";
  html += "          if (log.length > 0) {\n";
  html += "            tableBody.innerHTML += '<tr><td>' + log + '</td></tr>';\n";
  html += "          }\n";
  html += "        });\n";
  html += "      });\n";
  html += "  }\n";
  
  html += "  window.onload = function() {\n";
  html += "    updateIRLogs();\n";
  html += "    setInterval(updateIRLogs, 500); // Poll twice a second\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize IR Receiver module
  IrReceiver.begin(IR_RECEIVE_PIN, ENABLE_LED_FEEDBACK);
  
  Serial.println("\nESP32 IR Receiver Station Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/ir", handleGetIRData);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addIRLog("IR Receiver Armed");
}

void loop() {
  server.handleClient();
  
  // 1. Check if an IR signal has been received and decoded
  if (IrReceiver.decode()) {
    // 2. Extract code details
    uint16_t address = IrReceiver.decodedIRData.address;
    uint16_t command = IrReceiver.decodedIRData.command;
    String protocol = getProtocolLabel(IrReceiver.decodedIRData.protocol);
    
    // Ignore repeat codes (usually command = 0 in some protocols)
    if (!(IrReceiver.decodedIRData.flags & IRDATA_FLAGS_IS_REPEAT)) {
      char hexString[30];
      sprintf(hexString, "CMD: 0x%04X (Addr: 0x%04X) [%s]", command, address, protocol.c_str());
      
      addIRLog(String(hexString));
    }
    
    // 3. Resume receiving infrared signals
    IrReceiver.resume();
  }
  
  delay(1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **IR Receiver**, and **IR Remote** onto the canvas.
2. Wire the IR Receiver's output to **GPIO14**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. Click any button on the simulated IR Remote widget. Verify that the decoded command code displays in the web logs list.

## Expected Output
Serial Monitor:
```
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[IR DECODER] [0s] IR Receiver Armed
[IR DECODER] [8s] CMD: 0x00A2 (Addr: 0x0000) [NEC]
```

## Expected Canvas Behavior
* Clicking the simulated IR remote button widget fires infrared pulses, instantly updating the web dashboard table.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `IrReceiver.begin(...)` | Configures the receiver pin and starts decoding. |
| `IrReceiver.decode()` | Returns true when a complete infrared command is decoded. |
| `IrReceiver.decodedIRData.command` | Retrieves the decoded button command value. |
| `IrReceiver.resume()` | Resets the receiver registers to listen for the next pulse stream. |

## Hardware & Safety Concept: IR Signal Multiplexing and Carrier Frequencies
* **Carrier Frequency**: Consumer remote controls modulate infrared light at a carrier frequency (usually 38 kHz) to prevent ambient light (such as sunlight or lightbulbs) from triggering false inputs. TSOP38238 sensors feature an internal bandpass filter tuned to 38 kHz to filter out this noise.
* **Non-blocking loops**: Infrared pulse decoding requires microsecond-level timing. Avoid calling `delay()` in the main loop to prevent dropping command pulses.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and print the decoded hex command value locally.
2. **Relay Actuation**: Add a relay (GPIO 13) and toggle it when button `0x00A2` (Power button) is clicked.
3. **Sound Confirmation**: Add a buzzer (GPIO 15) and sound a short chirp when a valid IR command is decoded.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Receiver does not decode | Pin mapping wrong | Confirm that the receiver signal output is connected to GPIO 14 |
| Repeats register as new buttons | Flag filter missing | Verify that repeat codes are filtered out by checking `IRDATA_FLAGS_IS_REPEAT` in code |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [30 - Laser receiver wireless gate trigger](../beginner/30-laser-receiver-wireless-gate-trigger.md)
- [116 - ESP32 LED Bar Graph Level Indicator IoT](116-esp32-led-bar-graph-level-indicator-iot.md)
- [118 - ESP32 IR Remote Web LED Control](118-esp32-ir-remote-web-led-control.md) (Next project)
