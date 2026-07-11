# 189 - Email SMTP alert sender (Direct SSL mail client logs from ESP32)

Build an automated hazard notifier on the ESP32 that monitors an MQ-2 gas/smoke sensor on GPIO 14, sounds a buzzer alarm on GPIO 15, and sends secure SMTP email alerts directly to your inbox using SSL port 465.

## Goal
Learn how to configure SMTP mail client daemons, establish secure SSL socket handshakes, compose formatted email messages, set up secure app-specific authentication passwords, and manage threshold alarms.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An MQ-2 smoke sensor is wired to GPIO 14. If the gas concentration crosses a predefined safety threshold (e.g., 25%), the ESP32 triggers a local buzzer on GPIO 15, flashes an LED on GPIO 12, and sends an email ("⚠️ HAZARD WARNING: Smoke detected!") to the administrator's email inbox via a secure Gmail/Outlook SMTP server connection.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| MQ-2 Gas/Smoke Sensor | `mq2` | Yes | Yes |
| Active/Passive Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | AO (Analog Out) | GPIO14 | Yellow | Gas sensor analog input |
| Buzzer | Positive (+) | GPIO15 | Blue | Alarm sounder |
| LED | Anode (+) | GPIO12 | Red | Visual alert flash |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED, and a 100 Ω resistor with the buzzer.

## Code
```cpp
// Email SMTP Alert Sender (MQ-2 analog watch + SSL Port 465 + ESP Mail Client + Email Cooldown)
#include <WiFi.h>
#include <ESP_Mail_Client.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// SMTP Server Configuration (Example using Gmail SMTP)
#define SMTP_HOST "smtp.gmail.com"
#define SMTP_PORT 465 // SSL Port

// Sender Credentials (Replace with your actual email and App Password)
#define SENDER_EMAIL "myaddress@gmail.com"
#define SENDER_PASSWORD "xxxx xxxx xxxx xxxx" // App Password (not standard account password)

// Recipient Email
#define RECIPIENT_EMAIL "admin@example.com"

const int MQ2_PIN = 14;
const int BUZZER_PIN = 15;
const int LED_PIN = 12;

// Safety Threshold (out of 100%)
const float GAS_ALERT_THRESHOLD_PCT = 25.0;

// Cooldown tracker (prevents flooding inbox with emails)
unsigned long lastEmailTime = 0;
const unsigned long EMAIL_COOLDOWN_MS = 300000; // 5-minute email cooldown

// SMTP Session object
SMTPSession smtp;

// Callback function to monitor SMTP transmission status
void smtpCallback(SMTP_Status status) {
  Serial.println(status.info());
  if (status.success()) {
    Serial.println("[SMTP] Email sent successfully!");
  }
}

// Send alert email using SMTP SSL connection
void sendAlertEmail(float gasValue) {
  Serial.println("[SMTP] Preparing security alert email...");
  
  ESP_Mail_Session session;
  session.server.host_name = SMTP_HOST;
  session.server.port = SMTP_PORT;
  session.login.user_name = SENDER_EMAIL;
  session.login.password = SENDER_PASSWORD;
  session.login.user_name = SENDER_EMAIL;
  
  SMTP_Message message;
  message.sender.name = "ESP32 Security Station";
  message.sender.email = SENDER_EMAIL;
  message.subject = "⚠️ HAZARD ALERT: Smoke/Gas Breach Detected!";
  message.addRecipient("Admin", RECIPIENT_EMAIL);
  
  // HTML format message body
  String htmlMsg = "<div style='font-family:sans-serif;border:1px solid #ef4444;padding:20px;border-radius:8px;'>";
  htmlMsg += "<h2 style='color:#ef4444;'>Security Threshold Breached</h2>";
  htmlMsg += "<p>The MQ-2 sensor has detected high concentrations of smoke or gas.</p>";
  htmlMsg += "<ul>";
  htmlMsg += "  <li><strong>Sensor Reading:</strong> " + String(gasValue, 1) + "%</li>";
  htmlMsg += "  <li><strong>Uptime:</strong> " + String(millis() / 1000) + " seconds</li>";
  htmlMsg += "</ul>";
  htmlMsg += "<p style='color:#64748b;font-size:12px;'>This is an automated message sent from the ESP32 IoT Node.</p>";
  htmlMsg += "</div>";
  
  message.html.content = htmlMsg.c_str();
  message.html.charSet = "utf-8";
  message.html.transfer_encoding = Content_Transfer_Encoding::enc_7bit;
  
  // Connect and transmit
  if (!smtp.connect(&session)) {
    Serial.println("[SMTP Error] Connection failed!");
    return;
  }
  
  if (!MailClient.sendMail(&smtp, &message)) {
    Serial.println("[SMTP Error] Sending failed! " + smtp.errorReason());
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 SMTP Alarm Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Initialize SMTP session callback
  smtp.callback(smtpCallback);
}

void loop() {
  // Read analog gas sensor value
  int rawVal = analogRead(MQ2_PIN);
  float gasPct = (rawVal / 4095.0) * 100.0;
  
  static unsigned long lastLog = 0;
  if (millis() - lastLog > 2000) {
    lastLog = millis();
    Serial.printf("[Sensor] Gas level: %.1f%%\n", gasPct);
  }
  
  // Check if gas concentration crosses safety threshold
  if (gasPct >= GAS_ALERT_THRESHOLD_PCT) {
    // Sound siren locally
    digitalWrite(LED_PIN, HIGH);
    tone(BUZZER_PIN, 1200); // 1.2 kHz alarm pitch
    
    unsigned long now = millis();
    if (now - lastEmailTime >= EMAIL_COOLDOWN_MS || lastEmailTime == 0) {
      lastEmailTime = now;
      
      // Stop alarm briefly to perform SSL connection (avoids delay stuttering)
      noTone(BUZZER_PIN);
      
      // Send secure SMTP email
      sendAlertEmail(gasPct);
    }
  } else {
    digitalWrite(LED_PIN, LOW);
    noTone(BUZZER_PIN);
  }
  
  delay(50);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **MQ-2 Sensor**, **Buzzer**, and **LED** onto the canvas.
2. Wire MQ-2 AO to **GPIO14**, Buzzer to **GPIO15**, and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated MQ-2 gas concentration slider to 40% (exceeding the 25% threshold).
5. Verify that the buzzer sounds, the LED turns ON, and the console shows the SMTP connection attempt.
6. In simulation, since mail server connections are mocked, verify that the log displays the simulated email payload and confirms delivery.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[Sensor] Gas level: 5.0%
[Sensor] Gas level: 40.0%
[SMTP] Preparing security alert email...
Connecting to SMTP server...
[SMTP] Email sent successfully!
[Sensor] Gas level: 40.0%
```

## Expected Canvas Behavior
* Toggling the simulated MQ-2 slider above 25% starts the buzzer widget, turns ON the LED widget, and writes the email payload to the serial console.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <ESP_Mail_Client.h>` | Includes the SMTP/IMAP client engine for ESP32. |
| `session.server.port = 465` | Configures the SMTP client to use secure SSL port 465. |
| `message.html.content` | Binds the HTML formatted string to the email body layout. |
| `MailClient.sendMail(...)` | Connects, performs SSL handshake, authenticates, and sends the email. |

## Hardware & Safety Concept: App Passwords and SMTP Connection Blocking
* **App Passwords**: Major mail providers (like Gmail and Outlook) block direct logins using your account password to prevent credential theft. To send emails from the ESP32, you must enable Two-Factor Authentication on your account and generate a dedicated 16-character **App Password** to use in the code.
* **SMTP Blocking**: Some public internet service providers (ISPs) block standard outgoing SMTP port 25 or 465 to prevent spam botnets. If your connection fails repeatedly, ensure your network adapter or router permits outgoing connections on port 465/587.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "SMTP Status: Sent".
2. **Dynamic Recipient Update**: Modify the code to change the recipient email address via a local web configuration form (Project 172).
3. **Double Alert fall-back**: Combine this project with Project 188 to send both a Telegram alert and an email alert when a breach occurs.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with authentication error | Incorrect App Password | Ensure you are using a generated App Password, not your standard account password, and check for typos |
| Connection times out | Port blocked by ISP | Try changing the SMTP port to `587` (TLS) or check your router settings to ensure port 465 is open |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [98 - Gas Leakage Alarm IoT Node](../intermediate/98-gas-leakage-alarm-iot-node.md)
- [148 - IFTTT Alarm Notification node](../advanced/148-ifttt-alarm-notification-node.md)
- [190 - NTP Calendar Clock with Web Scheduler](190-ntp-calendar-clock-with-web-scheduler.md) (Next project)
