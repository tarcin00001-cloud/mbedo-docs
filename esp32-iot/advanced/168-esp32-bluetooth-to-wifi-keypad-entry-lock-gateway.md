# 168 - Bluetooth to WiFi Keypad Entry Lock Gateway

Build a secure hybrid gateway lock system on the ESP32 that receives entry commands via a local 4x4 matrix keypad or Bluetooth Classic serial requests from a smartphone, bridges validation logs to a remote cloud database using WiFi HTTP POST requests, and actuates a lock servo on GPIO 2.

## Goal
Learn how to run hybrid radio architectures (Bluetooth Classic and WiFi STA simultaneously), process matrix keypad entry buffers, construct HTTP Client payloads, and control servo mechanisms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and starts a Bluetooth Classic serial interface named `ESP32-Lock-Gateway`. A 4x4 Keypad is wired on GPIOs, and a lock servo on GPIO 2. Entering the correct PIN (`1234`) on the keypad or sending the character `U` over Bluetooth Classic triggers the gateway. The ESP32 locks/unlocks the servo, prints status to a local web server dashboard, and posts an entry validation log to a remote cloud database (`http://httpbin.org/post`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Rows | R1 - R4 | GPIO12 / 13 / 25 / 26 | Blue / White | Row outputs |
| Keypad Cols | C1 - C4 | GPIO27 / 32 / 33 / 15 | Purple / Grey | Column inputs |
| Servo Motor | PWM (Signal) | GPIO2 | Yellow | Latch controller |
| Buzzer | Positive (+) | GPIO4 | Red | Audio feedback |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the servo power to the 5V Vin pin and the buzzer to GPIO 4.

## Code
```cpp
// Bluetooth to WiFi Keypad Entry Lock Gateway (Bluetooth Classic + WiFi HTTP POST + Keypad + Servo)
#include <WiFi.h>
#include <WebServer.h>
#include <HTTPClient.h>
#include <BluetoothSerial.h>
#include <Keypad.h>
#include <ESP32Servo.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Cloud DB API endpoint
const char* cloudApiUrl = "http://httpbin.org/post";

// Bluetooth Classic Serial Object
BluetoothSerial SerialBT;

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 13, 25, 26};
byte colPins[COLS] = {27, 32, 33, 15};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

const int SERVO_PIN = 2;
const int BUZZER_PIN = 4;
Servo lockServo;

WebServer server(80);

// Lock logic variables
const String AUTH_PIN = "1234";
String pinBuffer = "";
bool isLocked = true;
unsigned long unlockTime = 0;
const unsigned long AUTO_RELOCK_DELAY_MS = 3000;

// Log buffer (Stores last 5 entries)
String gatewayLogs[5] = {"", "", "", "", ""};

void addLog(String message) {
  for (int i = 4; i > 0; i--) {
    gatewayLogs[i] = gatewayLogs[i - 1];
  }
  unsigned long timestampS = millis() / 1000;
  gatewayLogs[0] = "[" + String(timestampS) + "s] " + message;
  Serial.println("[Gateway] " + gatewayLogs[0]);
}

void soundBeep(int freq, int duration) {
  tone(BUZZER_PIN, freq);
  delay(duration);
  noTone(BUZZER_PIN);
}

// Perform lock transition
void performLock() {
  isLocked = true;
  lockServo.write(0); // LOCKED angle
  addLog("Gate Auto-Locked");
  soundBeep(800, 200);
}

// Perform unlock and bridge validation data to remote cloud
void performUnlock(String unlockSource) {
  isLocked = false;
  unlockTime = millis();
  lockServo.write(90); // UNLOCKED angle
  addLog("Gate Unlocked via " + unlockSource);
  soundBeep(1200, 150);
  
  // Bridge event to cloud database via WiFi STA HTTP client
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    http.begin(cloudApiUrl);
    http.addHeader("Content-Type", "application/json");
    
    String payload = "{\"device\":\"ESP32-Gateway-01\",\"event\":\"UNLOCK\",\"source\":\"" + unlockSource + "\",\"timestamp\":" + String(millis() / 1000) + "}";
    
    int httpResponseCode = http.POST(payload);
    Serial.printf("[Cloud Bridge] POST response code: %d\n", httpResponseCode);
    http.end();
  } else {
    Serial.println("[Cloud Bridge Error] WiFi offline. Saved locally only.");
  }
}

// HTTP API endpoint returning JSON data
void handleGetStatus() {
  String json = "{\"locked\":" + String(isLocked ? "true" : "false") + 
                 ",\"logs\":[";
  for (int i = 0; i < 5; i++) {
    json += "\"" + gatewayLogs[i] + "\"";
    if (i < 4) json += ",";
  }
  json += "]}";
  server.send(200, "application/json", json);
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Gateway Controller</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 450px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-top: 0; }\n";
  html += "  .status-badge { display: inline-block; padding: 10px 20px; border-radius: 20px; font-weight: bold; font-size: 15px; text-transform: uppercase; margin: 20px 0; background-color: #7f1d1d; color: #fca5a5; }\n";
  html += "  .status-badge.unlocked { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  table { width: 100%; border-collapse: collapse; margin-top: 25px; text-align: left; font-size: 13px; }\n";
  html += "  th { color: #64748b; border-bottom: 2px solid #334155; padding-bottom: 8px; }\n";
  html += "  td { padding: 8px 0; border-bottom: 1px solid #1e293b; color: #94a3b8; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Lock Gateway Status</h1>\n";
  html += "  <div id=\"statusBadge\" class=\"status-badge\">LOCKED</div>\n";
  
  html += "  <h3>Entry Logs (WiFi Bridged)</h3>\n";
  html += "  <table>\n";
  html += "    <thead><tr><th>Log Entry</th></tr></thead>\n";
  html += "    <tbody id=\"logTable\"></tbody>\n";
  html += "  </table>\n";
  
  html += "  <p class=\"footer\">Bluetooth Classic: ESP32-Lock-Gateway | Port 80</p>\n";
  html += "</div>\n";
  
  // JavaScript AJAX loop
  html += "<script>\n";
  html += "  function updateHUD() {\n";
  html += "    fetch('/api/status')\n";
  html += "      .then(response => response.json())\n";
  html += "      .then(data => {\n";
  html += "        const bdgEl = document.getElementById('statusBadge');\n";
  if ("data.locked") {\n";
  html += "          bdgEl.innerText = 'LOCKED';\n";
  html += "          bdgEl.className = 'status-badge';\n";
  html += "        } else {\n";
  html += "          bdgEl.innerText = 'UNLOCKED';\n";
  html += "          bdgEl.className = 'status-badge unlocked';\n";
  html += "        }\n";
  
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
  html += "    updateHUD();\n";
  html += "    setInterval(updateHUD, 1000);\n";
  html += "  };\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize Bluetooth Classic
  SerialBT.begin("ESP32-Lock-Gateway");
  Serial.println("Bluetooth Classic active as 'ESP32-Lock-Gateway'");
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // CLOSED/LOCKED position
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 Lock Gateway booting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  server.on("/", handleRoot);
  server.on("/api/status", handleGetStatus);
  
  server.begin();
  Serial.print("HTTP Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  addLog("Lock Gateway Online");
}

void loop() {
  server.handleClient();
  
  // 1. Check Bluetooth Classic serial commands
  if (SerialBT.available()) {
    char btCmd = SerialBT.read();
    Serial.printf("[Bluetooth] Command received: %c\n", btCmd);
    
    if (btCmd == 'U' || btCmd == 'u') {
      performUnlock("Bluetooth Smartphone App");
    } else if (btCmd == 'L' || btCmd == 'l') {
      performLock();
    } else {
      soundBeep(400, 300); // Invalid Bluetooth command
    }
  }
  
  // 2. Check Physical 4x4 Keypad entries
  char key = keypad.getKey();
  if (key != NO_KEY) {
    soundBeep(1000, 50);
    
    if (key == '*') {
      pinBuffer = "";
      addLog("PIN entry cleared");
      soundBeep(400, 300);
    } 
    else {
      pinBuffer += key;
      if (pinBuffer.length() >= 4) {
        if (pinBuffer == AUTH_PIN) {
          performUnlock("Local Keypad PIN");
        } else {
          pinBuffer = "";
          addLog("Incorrect PIN entered");
          soundBeep(400, 500);
        }
      }
    }
  }
  
  // 3. Automated Auto-Relock routine
  if (!isLocked && (millis() - unlockTime >= AUTO_RELOCK_DELAY_MS)) {
    performLock();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **4x4 Keypad**, **Servo Motor**, and **Buzzer** onto the canvas.
2. Wire Keypad rows to **GPIO12, 13, 25, 26** and columns to **GPIO27, 32, 33, 15**.
3. Wire Servo to **GPIO2** and Buzzer to **GPIO4**.
4. Paste the code and click **Run**.
5. Open your web browser and navigate to the printed IP address.
6. Click digits `1`, `2`, `3`, and `4` sequentially on the simulated keypad. Verify that the servo rotates to 90 degrees and the success chime plays.
7. Open the serial console. Enter the command `U` to simulate receiving an unlock command via Bluetooth. Verify that the servo rotates to 90 degrees and a POST request is sent to `httpbin.org`.

## Expected Output
Serial Monitor:
```
Bluetooth Classic active as 'ESP32-Lock-Gateway'
WiFi Connected.
HTTP Server active. Connect at: http://10.10.0.3
[Gateway] [0s] Lock Gateway Online
[Gateway] [12s] Gate Unlocked via Local Keypad PIN
[Cloud Bridge] POST response code: 200
[Gateway] [15s] Gate Auto-Locked
```

## Expected Canvas Behavior
* Pressing the correct keypad combination rotates the servo latch widget and triggers buzzer alarm chimes.
* Entering `U` in the serial terminal simulates Bluetooth unlock commands, triggering the servo.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SerialBT.begin("ESP32-Lock-Gateway")` | Starts the Bluetooth Classic serial broadcast server. |
| `SerialBT.read()` | Non-blocking query checking for incoming Bluetooth serial data. |
| `http.POST(payload)` | Bridges the entry event telemetry to the remote cloud server database over WiFi. |
| `lockServo.write(90)` | Steps the servo lock latch. |

## Hardware & Safety Concept: RF Interference and Coexistence
* **RF Coexistence**: ESP32 uses the same physical 2.4 GHz internal radio antenna for both WiFi and Bluetooth. Using both protocols simultaneously can lead to resource contention, slowing down web requests or causing Bluetooth audio crackling/disconnects. In high-traffic environments, manage coexistence profiles in the ESP-IDF framework or space out transmission loops.
* **Buzzer Chimes**: Passive piezo buzzers require hardware timers. In ESP32, using `tone()` alongside `ledc` wrappers is safe as the core provides 16 LEDC channels.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a Bluetooth logo when a client is connected.
2. **Alert webhook**: Trigger an IFTTT email notification webhook (Project 148) if an incorrect PIN is entered 3 times in a row.
3. **SPIFFS configuration**: Load the authorized PIN from a file in SPIFFS (Project 165).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bluetooth device not showing up on phone scan | Antenna shield or BLE conflict | Ensure you are scanning for classic Bluetooth, not BLE (Bluetooth Low Energy), on your phone. Classic Bluetooth requires pairing in the phone's system settings |
| WiFi drops out when Bluetooth is active | RF coexistence conflict | Place a 100 ms delay between Bluetooth transactions to give the internal radio time to switch back to WiFi mode |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 password lock with access logs](114-esp32-keypad-4x4-password-lock-iot-keypad-servo-web-access-logs.md)
- [143 - Firebase Authentication Access Controller](143-firebase-authentication-access-controller.md)
- [165 - Smart Door Lock with Web Access Override logs](165-esp32-smart-door-lock-with-web-access-override-logs.md)
- [169 - Multi-zone Temperature logger IoT](169-esp32-multi-zone-temperature-logger-iot-3x-ds18b20-cloud-databases.md) (Next project)
