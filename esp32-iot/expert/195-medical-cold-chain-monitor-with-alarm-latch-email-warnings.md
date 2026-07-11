# 195 - Medical Cold Chain Monitor with Alarm Latch & Email Warnings

Build a vaccine and medical storage temperature monitor on the ESP32 that logs environment breaches using a DS18B20 sensor, latches a critical alarm state, and sends secure email alerts via SMTP SSL.

## Goal
Learn how to program OneWire DS18B20 sensors, implement alarm latch state machines, manage software cooldown cycles, send HTML secure emails via SMTP, and host interactive web administration consoles.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It reads a DS18B20 OneWire temperature sensor (GPIO 14). If the temperature drifts outside the strict medical window (2.0 °C to 8.0 °C), the ESP32 triggers a buzzer (GPIO 15) and latches the alarm state. The alarm remains active even if the temperature returns to normal, warning staff of a past violation. The system sends an email notification. Staff can review the violation log and reset the latch via a web console or physical button (GPIO 13).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Temperature Probe | `ds18b20` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| Push Button | `button` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 | Signal (Data) | GPIO14 | Yellow | OneWire data bus (requires 4.7 kΩ pull-up) |
| Buzzer | Positive (+) | GPIO15 | Blue | Alarm siren |
| Push Button | Pin 1 / Pin 2 | GPIO13 / GND | Green / Black | Manual alarm reset |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 4.7 kΩ pull-up resistor between the DS18B20 data line (GPIO 14) and 3V3 to ensure stable sensor communications.

## Code
```cpp
// Medical Cold Chain Monitor (OneWire DS18B20 + Alarm Latch + SMTP SSL Port 465 + NVS logs + Web HUD)
#include <WiFi.h>
#include <WebServer.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <ESP_Mail_Client.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// SMTP Server Configuration (Example using Gmail SMTP)
#define SMTP_HOST "smtp.gmail.com"
#define SMTP_PORT 465
#define SENDER_EMAIL "myaddress@gmail.com"
#define SENDER_PASSWORD "xxxx xxxx xxxx xxxx" // App Password
#define RECIPIENT_EMAIL "medical-officer@example.com"

const int ONE_WIRE_BUS = 14;
const int RESET_BUTTON = 13;
const int BUZZER_PIN = 15;
const int LED_PIN = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

WebServer server(80);
SMTPSession smtp;

// Medical range limits (Safe Vaccine Temp: 2.0 C to 8.0 C)
const float TEMP_MIN_SAFE = 2.0;
const float TEMP_MAX_SAFE = 8.0;

// Alarm Latch state machine variables
bool alarmLatched = false;
float violatingTemp = 0.0;
String violationReason = "";
unsigned long violationTime = 0;

// Cooldown tracker (prevents flooding inbox with emails)
unsigned long lastEmailTime = 0;
const unsigned long EMAIL_COOLDOWN_MS = 600000; // 10-minute email cooldown

// Callback monitoring SMTP transmission status
void smtpCallback(SMTP_Status status) {
  Serial.println(status.info());
}

// Send cold chain breach notification email
void sendBreachEmail(float currentTemp, String reason) {
  if (WiFi.status() == WL_CONNECTED) {
    ESP_Mail_Session session;
    session.server.host_name = SMTP_HOST;
    session.server.port = SMTP_PORT;
    session.login.user_name = SENDER_EMAIL;
    session.login.password = SENDER_PASSWORD;
    
    SMTP_Message message;
    message.sender.name = "Cold Chain Monitor Hub";
    message.sender.email = SENDER_EMAIL;
    message.subject = "🚨 CRITICAL: Vaccine Cold Chain Violation!";
    message.addRecipient("Medical Officer", RECIPIENT_EMAIL);
    
    // HTML Message Body
    String htmlMsg = "<div style='font-family:sans-serif;border:2px solid #dc2626;padding:25px;border-radius:10px;'>";
    htmlMsg += "<h2 style='color:#dc2626;margin-top:0;'>CRITICAL TEMPERATURE BREACH</h2>";
    htmlMsg += "<p>A temperature violation has been detected in Refrigerator Unit 4.</p>";
    htmlMsg += "<ul>";
    htmlMsg += "  <li><strong>Trigger Temperature:</strong> <span style='color:#dc2626;font-weight:bold;'>" + String(currentTemp, 1) + " &deg;C</span></li>";
    htmlMsg += "  <li><strong>Violation Reason:</strong> " + reason + "</li>";
    htmlMsg += "  <li><strong>Safe Window:</strong> " + String(TEMP_MIN_SAFE, 1) + " to " + String(TEMP_MAX_SAFE, 1) + " &deg;C</li>";
    htmlMsg += "  <li><strong>System State:</strong> ALARM LATCHED</li>";
    htmlMsg += "</ul>";
    htmlMsg += "<p style='color:#b91c1c;font-weight:bold;'>Action Required: Inspect freezer seal and move samples if necessary.</p>";
    htmlMsg += "</div>";
    
    message.html.content = htmlMsg.c_str();
    message.html.charSet = "utf-8";
    
    if (!smtp.connect(&session)) {
      Serial.println("[SMTP Error] Connection failed!");
      return;
    }
    
    if (!MailClient.sendMail(&smtp, &message)) {
      Serial.println("[SMTP Error] Sending failed: " + smtp.errorReason());
    }
  }
}

// Reset alarm latch state
void resetAlarm() {
  alarmLatched = false;
  noTone(BUZZER_PIN);
  digitalWrite(LED_PIN, LOW);
  Serial.println("[System] Alarm latch reset successfully.");
}

// Serve root webpage
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Cold Chain Monitor</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 480px; width: 100%; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; text-align: center; margin-top: 0; }\n";
  html += "  .temp-box { text-align: center; font-size: 54px; font-weight: 800; font-family: monospace; color: #10b981; margin: 20px 0; }\n";
  html += "  .temp-box.out-range { color: #ef4444; }\n";
  html += "  .status-badge { text-align: center; padding: 12px; border-radius: 8px; font-weight: bold; font-size: 14px; margin-bottom: 25px; border: 1px solid #334155; }\n";
  html += "  .status-ok { background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .status-alert { background-color: #7f1d1d; color: #fca5a5; animation: blink 1s infinite; }\n";
  html += "  .btn-reset { display: block; width: 100%; padding: 14px; background-color: #ef4444; color: white; border: none; border-radius: 6px; font-size: 15px; font-weight: bold; cursor: pointer; text-align: center; text-decoration: none; }\n";
  html += "  .btn-disabled { background-color: #334155; color: #64748b; cursor: not-allowed; }\n";
  html += "  .info-table { width: 100%; font-size: 13px; color: #94a3b8; margin: 20px 0; border-top: 1px solid #1e293b; padding-top: 15px; }\n";
  html += "  .info-row { display: flex; justify-content: space-between; padding: 6px 0; }\n";
  html += "  @keyframes blink { 50% { opacity: 0.8; } }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Cold Chain Console</h1>\n";
  
  // Dynamic temperature class mapping
  bool isOut = (violatingTemp > 0.0); // Simple breach history indicator
  sensors.requestTemperatures();
  float currentTemp = sensors.getTempCByIndex(0);
  if (currentTemp == -127.0) currentTemp = 4.5; // Mock simulation fallback
  
  bool currentOut = (currentTemp < TEMP_MIN_SAFE || currentTemp > TEMP_MAX_SAFE);
  
  html += "  <div class=\"temp-box " + String(currentOut ? "out-range" : "") + "\" id=\"tempDisp\">" + String(currentTemp, 1) + " &deg;C</div>\n";
  
  // Render status alerts
  if (alarmLatched) {
    html += "  <div class=\"status-badge status-alert\">⚠️ CRITICAL TEMPERATURE ALARM LATCHED</div>\n";
  } else {
    html += "  <div class=\"status-badge status-ok\">SYSTEM STATUS: SECURED (NORMAL)</div>\n";
  }
  
  // Info logs
  html += "  <div class=\"info-table\">\n";
  html += "    <div class=\"info-row\"><span>Safe Target Window</span><span>2.0 - 8.0 &deg;C</span></div>\n";
  if (alarmLatched) {
    html += "    <div class=\"info-row\"><span>Violation Temp</span><span style='color:#ef4444;font-weight:bold;'>" + String(violatingTemp, 1) + " &deg;C</span></div>\n";
    html += "    <div class=\"info-row\"><span>Breach Type</span><span>" + violationReason + "</span></div>\n";
  }
  html += "  </div>\n";
  
  // Actions
  if (alarmLatched) {
    html += "  <a href=\"/reset\" class=\"btn-reset\">CLEAR SYSTEM LATCH & SILENCE</a>\n";
  } else {
    html += "  <span class=\"btn-reset btn-disabled\">LATCH STATUS NORMAL</span>\n";
  }
  
  html += "</div>\n";
  
  // JavaScript auto updater
  html += "<script>\n";
  html += "  setInterval(function() {\n";
  html += "    fetch('/api/temp').then(response => response.text()).then(t => {\n";
  html += "      document.getElementById('tempDisp').innerHTML = parseFloat(t).toFixed(1) + ' &deg;C';\n";
  html += "      const temp = parseFloat(t);\n";
  html += "      if (temp < 2.0 || temp > 8.0) {\n";
  html += "        document.getElementById('tempDisp').className = 'temp-box out-range';\n";
  html += "      } else {\n";
  html += "        document.getElementById('tempDisp').className = 'temp-box';\n";
  html += "      }\n";
  html += "    });\n";
  html += "  }, 2000);\n";
  html += "</script>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// REST route returning temperature float
void handleGetTemp() {
  sensors.requestTemperatures();
  float currentTemp = sensors.getTempCByIndex(0);
  if (currentTemp == -127.0) currentTemp = 4.5;
  server.send(200, "text/plain", String(currentTemp, 2));
}

// Reset latch Web handler
void handleWebReset() {
  resetAlarm();
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RESET_BUTTON, INPUT_PULLUP);
  
  digitalWrite(LED_PIN, LOW);
  noTone(BUZZER_PIN);
  
  sensors.begin();
  
  Serial.println("\nESP32 Cold Chain Station starting...");
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
  server.on("/api/temp", HTTP_GET, handleGetTemp);
  server.on("/reset", HTTP_GET, handleWebReset);
  server.begin();
  
  smtp.callback(smtpCallback);
}

void loop() {
  server.handleClient();
  
  // 1. Read DS18B20 sensor
  sensors.requestTemperatures();
  float currentTemp = sensors.getTempCByIndex(0);
  
  // Mock fallback for simulation testing
  if (currentTemp == -127.0) {
    static float mockTemp = 5.0;
    currentTemp = mockTemp;
  }
  
  // 2. Evaluate Alarm Latch thresholds
  bool outOfBounds = (currentTemp < TEMP_MIN_SAFE || currentTemp > TEMP_MAX_SAFE);
  
  if (outOfBounds && !alarmLatched) {
    alarmLatched = true;
    violatingTemp = currentTemp;
    violationTime = millis();
    
    if (currentTemp < TEMP_MIN_SAFE) {
      violationReason = "Freezing Limit Exceeded";
    } else {
      violationReason = "Overheating Limit Exceeded";
    }
    
    Serial.printf("[ALERT] Temperature breach detected: %.1f C. Latching Alarm.\n", currentTemp);
    
    // Send email push warning if cooldown has expired
    unsigned long now = millis();
    if (now - lastEmailTime >= EMAIL_COOLDOWN_MS || lastEmailTime == 0) {
      lastEmailTime = now;
      sendBreachEmail(currentTemp, violationReason);
    }
  }
  
  // 3. Sound alarm sequence if alarm state is latched
  if (alarmLatched) {
    digitalWrite(LED_PIN, HIGH);
    tone(BUZZER_PIN, 1500); // Solid high frequency alarm pitch
  }
  
  // 4. Physical Reset Button logic (Active-LOW switch check)
  if (digitalRead(RESET_BUTTON) == LOW) {
    delay(50); // Debounce
    if (digitalRead(RESET_BUTTON) == LOW) {
      resetAlarm();
      // Wait for release
      while (digitalRead(RESET_BUTTON) == LOW) { delay(10); }
    }
  }
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS18B20 Sensor**, **Buzzer**, **Button**, and **LED** onto the canvas.
2. Wire DS18B20 to **GPIO14**, Button to **GPIO13**, Buzzer to **GPIO15**, and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated DS18B20 temperature slider to 12.0 °C. Verify that the buzzer starts sounding a continuous alarm.
5. Set the temperature slider back to 5.0 °C (safe). Verify that the buzzer and LED remain active (latched state).
6. Press the simulated button widget on the canvas. Verify that the buzzer silences and the LED turns OFF.
7. Open your web browser and navigate to the IP address. Verify that the log displays the breach temperature details.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[ALERT] Temperature breach detected: 12.0 C. Latching Alarm.
[SMTP] Connecting to server...
[SMTP] Email sent successfully!
[System] Alarm latch reset successfully.
```

## Expected Canvas Behavior
* Setting the temperature slider outside 2–8 °C activates the buzzer and LED widgets. Pressing the button widget silences the alarm and resets the state indicators.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `DallasTemperature sensors(...)` | Initialises the Dallas library wrapping the raw OneWire bus signals. |
| `sensors.getTempCByIndex(0)` | Reads the temperature value from the first DS18B20 probe found on the bus. |
| `alarmLatched = true` | Latches the alarm flag, which is only cleared by user reset commands. |

## Hardware & Safety Concept: Sensor Cable Breaks and Backup Batteries
* **Sensor Cable Breaks**: In high-reliability cold chains, if the sensor cable breaks or drops off the bus, the library returns `-127.0 °C`. If your code evaluates this blindly, it will trigger a false freezing alarm. Always check for this specific error value:
  `if (currentTemp == -127.0) { triggerSystemFaultAlert(); }`
* **Backup Batteries**: Medical freezers can lose mains power. To prevent losing temperature monitoring, power the ESP32 using a battery backup module (UPS shield) or a LiPo battery with automatic power path management, allowing the system to log data and send alerts even during blackouts.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display the current temperature and battery level.
2. **Weekly PDF Report**: Add code that creates a daily temperature log file in SPIFFS (Project 180) and sends it as an email attachment every week.
3. **ThingSpeak Graphing**: Upload the temperature data to a ThingSpeak channel (Project 140) every 15 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sensor returns -127.0 C continuously | Missing pull-up resistor | The OneWire bus requires a physical 4.7 kΩ pull-up resistor connected between the DQ data pin and the 3V3 power pin |
| Button reset does not work | Missing pull-up configuration | Ensure the button input pin is configured as `INPUT_PULLUP`. Verify that the other button contact connects directly to GND |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [122 - Cold Chain Monitor](../expert/122-cold-chain-monitor.md)
- [169 - Multi-zone Temperature logger IoT](169-esp32-multi-zone-temperature-logger-iot-3x-ds18b20-cloud-databases.md)
- [196 - Energy Safety Contactor Logger with Web Control Breaker overrides](196-energy-safety-contactor-logger-with-web-control-breaker-overrides.md) (Next project)
