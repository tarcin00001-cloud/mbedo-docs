# 188 - Telegram Bot Alarm Sender (Sensor breach → sends photo/text to chat)

Build an automated security notifier node on the ESP32 that monitors a PIR motion sensor on GPIO 14, sounds a local alert siren on GPIO 15, and pushes instant warning alerts to a Telegram chat over secure HTTPS.

## Goal
Learn how to configure interrupt-driven sensor inputs, execute unsolicited HTTPS notifications, handle push-state alerts, manage security cooldown timers, and interface buzzers.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A PIR motion sensor is wired to GPIO 14. When motion is detected, the ESP32 triggers a buzzer on GPIO 15, flashes a status LED on GPIO 12, and sends an encrypted HTTPS alert message ("🚨 MOTION DETECTED! Security Zone 1 breached!") directly to the owner's Telegram chat, implementing a 1-minute alert cooldown timer to prevent message flooding.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Active/Passive Buzzer | `buzzer` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| PIR Sensor | OUT (Signal) | GPIO14 | Yellow | Motion sensor input |
| Buzzer | Positive (+) | GPIO15 | Blue | Alarm sounder |
| LED | Anode (+) | GPIO12 | Red | Visual alert flash |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use a 220 Ω resistor in series with the LED, and a 100 Ω resistor with the buzzer.

## Code
```cpp
// Telegram Bot Alarm Sender (PIR sensor interrupt + Buzzer alarm + HTTPS push alert + Cooldown timer)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Telegram Bot Credentials (Replace with your actual Token and Chat ID)
const String botToken = "123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ";
const String authorizedChatId = "987654321"; // Your personal Chat ID

const int PIR_PIN = 14;
const int BUZZER_PIN = 15;
const int LED_PIN = 12;

// DigiCert Global Root CA (Used by api.telegram.org)
const char telegramRootCA[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIIDrzCCAp+gAwIBAgIQDfb1ZTyX7hSG8BSmSKsNfDANBgkqhkiG9w0BAQsFADBl\n"
"MQswCQYDVQQGEwJVUzEYMBYGA1UEChMPR2VvVHJ1c3QgSW5jLjEdMBsGA1UECxMU\n"
"R2VvVHJ1c3QgUHJpbWFyeSBDQSAxMTAwFw0wNjExMjcwMDAwMDBaFw0zMTExMTAw\n"
"MDAwMDBaMHMxCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAX\n"
"BgNVBAsTEHd3dy5kaWdpY2VydC5jb20xMS8wLQYDVQQDEyZEaWdpQ2VydCBCYWx0\n"
"aW1vcmUgQ3liZXJUcnVzdCBSb290WhgPMjEyNjA2MTcwNDMwMDBaMEUxCzAJBgNV\n"
"BAYTAlVTMQswCQYDVQQIDAJDQTEWMBQGA1UEBwwNU2FuIEZyYW5jaXNjbzEZMBcG\n"
"A1UECgwQQmFsdGltb3JlIENBMRcwFQYDVQQDDA5CYWx0aW1vcmUgUm9vdFwwDQYJ\n"
"KoZIhvcNAQEBBQADSwAwSAJBALN1wX6z5e2x+gVwU3hPUz2xXiMzsT5H3p85S+4X\n"
"uFv8/v53r9/Pzz/Pf79/Pz9/Pz9/Pz9/Pz9/Pz9/PwIDAQAB\n"
"-----END CERTIFICATE-----\n";

WiFiClientSecure wifiSecureClient;

// Cooldown tracker (prevents flooding user with messages)
unsigned long lastAlertTime = 0;
const unsigned long COOLDOWN_INTERVAL_MS = 60000; // 1-minute alert cooldown

volatile bool motionDetected = false;

// Interrupt Service Routine (ISR) triggered by PIR transition
void IRAM_ATTR handleMotionInterrupt() {
  motionDetected = true;
}

// Send alert push message to Telegram
void sendTelegramAlert(String messageText) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = "https://api.telegram.org/bot" + botToken + "/sendMessage";
    
    http.begin(wifiSecureClient, url);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    StaticJsonDocument<256> doc;
    doc["chat_id"] = authorizedChatId;
    doc["text"] = messageText;
    
    String jsonPayload;
    serializeJson(doc, jsonPayload);
    
    Serial.printf("[Telegram Alert] Sending push: %s\n", messageText.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode == 200) {
      Serial.println("[Telegram Alert] Push notification sent successfully.");
    } else {
      Serial.printf("[Telegram Error] Push failed. HTTP Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

// Execute alarm sound sequences
void triggerLocalAlarm() {
  Serial.println("[Security Node] Sounding local siren...");
  
  // Sound siren chimes
  for (int i = 0; i < 3; i++) {
    digitalWrite(LED_PIN, HIGH);
    tone(BUZZER_PIN, 1000); // 1 kHz pitch
    delay(200);
    digitalWrite(LED_PIN, LOW);
    noTone(BUZZER_PIN);
    delay(100);
    
    digitalWrite(LED_PIN, HIGH);
    tone(BUZZER_PIN, 800); // 800 Hz pitch
    delay(200);
    digitalWrite(LED_PIN, LOW);
    noTone(BUZZER_PIN);
    delay(100);
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  // Configure PIR input with internal pull-down and register interrupt
  pinMode(PIR_PIN, INPUT_PULLDOWN);
  attachInterrupt(digitalPinToInterrupt(PIR_PIN), handleMotionInterrupt, RISING);
  
  Serial.println("\nESP32 Telegram Alarm Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Set CA cert
  wifiSecureClient.setCACert(telegramRootCA);
  
  Serial.println("[Security Node] System armed and active.");
}

void loop() {
  // Check if motion was flagged by ISR and cooldown has expired
  if (motionDetected) {
    motionDetected = false; // Reset interrupt flag
    
    unsigned long now = millis();
    if (now - lastAlertTime >= COOLDOWN_INTERVAL_MS || lastAlertTime == 0) {
      lastAlertTime = now;
      
      // Sound alarm locally
      triggerLocalAlarm();
      
      // Send secure alert push to cloud
      sendTelegramAlert("🚨 SECURITY NOTICE: Motion detected in Zone 1!");
    } else {
      Serial.printf("[Security Node] Motion ignored (Cooldown: %d s remaining)\n", 
                    (COOLDOWN_INTERVAL_MS - (now - lastAlertTime)) / 1000);
    }
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **PIR Sensor**, **Buzzer**, and **LED** onto the canvas.
2. Wire PIR output to **GPIO14**, Buzzer to **GPIO15**, and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Click on the PIR sensor widget and click "Simulate Motion".
5. Verify that the buzzer plays the siren tones, the LED flashes, and the console displays the HTTPS alert push message.
6. Trigger the PIR sensor again within 10 seconds. Verify that the console logs the cooldown warning and skips sending the message.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[Security Node] System armed and active.
[Security Node] Sounding local siren...
[Telegram Alert] Sending push: 🚨 SECURITY NOTICE: Motion detected in Zone 1!
[Telegram Alert] Push notification sent successfully.
[Security Node] Motion ignored (Cooldown: 54 s remaining)
```

## Expected Canvas Behavior
* Triggering the PIR widget sounds the buzzer widget, blinks the LED widget, and writes the HTTPS payload status to the serial logs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `attachInterrupt(...)` | Registers the hardware level handler to run when the PIR pin goes HIGH. |
| `IRAM_ATTR` | Configures the interrupt function to execute from high-speed internal RAM. |
| `COOLDOWN_INTERVAL_MS` | Defines the duration the device must wait before sending another alert. |

## Hardware & Safety Concept: Interrupt Service Routines and Alarm Flood Protection
* **Interrupt Service Routines (ISRs)**: The PIR sensor transition triggers a hardware interrupt. ISR code must be extremely short and must **never** execute network calls, print statements, or delays. Set a volatile flag (`motionDetected`) and process the alarm logic in the main loop.
* **Alarm Flood Protection**: If a sensor develops a fault or a pet continuously triggers the PIR sensor, the ESP32 could send thousands of requests to the Telegram API, causing Telegram to block your bot's IP address. Always implement a software cooldown timer (minimum 1 minute) to enforce limits.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "System State: ARMED".
2. **Dynamic Cooldown Update**: Modify the code to change the cooldown timer value via a web slider panel (Project 166).
3. **Email SMTP Notification**: Add an email sender fallback (Project 189) that sends an email alert if the Telegram API fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No message received on phone | Incorrect Chat ID | Ensure you have sent `/start` to your bot from your personal account to initiate the chat, and ensure your chat ID string matches your actual ID |
| Buzzer does not sound | Incorrect tone pin | Passive buzzers require using the core `tone()` function on PWM capable pins. Verify the pin wiring and ensure the ground connection is shared |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [29 - Wireless PIR motion alarm receiver](../beginner/29-wireless-pir-motion-alarm-receiver.md)
- [187 - Telegram Bot Command Controller](187-telegram-bot-command-controller.md)
- [189 - Email SMTP alert sender](189-email-smtp-alert-sender.md) (Next project)
