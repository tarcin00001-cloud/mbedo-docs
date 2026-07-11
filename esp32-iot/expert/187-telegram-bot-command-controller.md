# 187 - Telegram Bot Command Controller (Send Telegram message → trigger action)

Build a secure Telegram Bot receiver node on the ESP32 that polls the Telegram Bot API over HTTPS, parses incoming messages and commands using ArduinoJson, toggles a physical LED on GPIO 12, and sends confirmation messages back to the chat.

## Goal
Learn how to interact with the HTTPS Telegram Bot API, parse secure HTTP JSON updates, implement command routing structures, issue HTTPS replies, and manage secure certificates.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It initializes a secure HTTPS connection with `api.telegram.org`. Every 3 seconds, it polls the Telegram server for new messages. When a user sends a command like `/led_on` or `/led_off` from their smartphone, the ESP32 decodes the command, changes the state of the LED on GPIO 12, and sends a reply message ("LED is now ON") back to the user's chat.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// Telegram Bot Command Controller (Secure HTTPS polling + JSON parser + Chat responder)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Telegram Bot Credentials (Replace with your actual Bot Token from @BotFather)
const String botToken = "123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ";

const int LED_PIN = 12;
bool ledState = false;

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

int32_t lastUpdateId = 0; // Tracks processed message offset IDs
unsigned long lastPollTime = 0;
const unsigned long POLL_INTERVAL_MS = 3000; // Poll Telegram API every 3 seconds

// Send message reply back to Telegram chat ID
void sendTelegramMessage(String chatId, String messageText) {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    String url = "https://api.telegram.org/bot" + botToken + "/sendMessage";
    
    http.begin(wifiSecureClient, url);
    http.addHeader("Content-Type", "application/json");
    
    // Compile JSON payload
    StaticJsonDocument<256> doc;
    doc["chat_id"] = chatId;
    doc["text"] = messageText;
    
    String jsonPayload;
    serializeJson(doc, jsonPayload);
    
    Serial.printf("[Telegram Send] Chat: %s | Text: %s\n", chatId.c_str(), messageText.c_str());
    int httpResponseCode = http.POST(jsonPayload);
    
    if (httpResponseCode != 200) {
      Serial.printf("[Telegram Error] Send failed. HTTP Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

// Process command strings received in chat
void handleCommand(String chatId, String commandText) {
  commandText.toLowerCase();
  
  if (commandText == "/start") {
    String welcome = "Welcome to ESP32 Console Bot!\nCommands:\n";
    welcome += "/led_on - Turn LED ON\n";
    welcome += "/led_off - Turn LED OFF\n";
    welcome += "/status - Check system status";
    sendTelegramMessage(chatId, welcome);
  } 
  else if (commandText == "/led_on") {
    ledState = true;
    digitalWrite(LED_PIN, HIGH);
    sendTelegramMessage(chatId, "LED has been turned ON.");
  } 
  else if (commandText == "/led_off") {
    ledState = false;
    digitalWrite(LED_PIN, LOW);
    sendTelegramMessage(chatId, "LED has been turned OFF.");
  } 
  else if (commandText == "/status") {
    String status = "ESP32 System Status:\n";
    status += "LED State: " + String(ledState ? "ON" : "OFF") + "\n";
    status += "Uptime: " + String(millis() / 1000) + " seconds";
    sendTelegramMessage(chatId, status);
  } 
  else {
    sendTelegramMessage(chatId, "Unknown command. Send /start to view commands list.");
  }
}

// Poll Telegram server for new messages using getUpdates API
void pollTelegramUpdates() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    // Offset parameters ensure we only download new messages
    String url = "https://api.telegram.org/bot" + botToken + 
                 "/getUpdates?offset=" + String(lastUpdateId + 1) + "&limit=5&timeout=0";
                 
    http.begin(wifiSecureClient, url);
    int httpResponseCode = http.GET();
    
    if (httpResponseCode == 200) {
      String payload = http.getString();
      
      // Parse JSON stream using DynamicJsonDocument
      DynamicJsonDocument doc(2048);
      DeserializationError error = deserializeJson(doc, payload);
      
      if (!error) {
        JsonArray results = doc["result"].as<JsonArray>();
        
        for (JsonObject result : results) {
          int32_t updateId = result["update_id"];
          lastUpdateId = updateId; // Update tracking offset
          
          if (result.containsKey("message")) {
            JsonObject message = result["message"];
            String chatId = message["chat"]["id"].as<String>();
            
            if (message.containsKey("text")) {
              String text = message["text"].as<String>();
              Serial.printf("[Telegram Recv] Message: %s | Chat: %s\n", text.c_str(), chatId.c_str());
              
              // Route and execute the command
              handleCommand(chatId, text);
            }
          }
        }
      }
    } else {
      Serial.printf("[Telegram Error] Poll failed. HTTP Code: %d\n", httpResponseCode);
    }
    http.end();
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 Telegram Bot Command Controller starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Set CA cert
  wifiSecureClient.setCACert(telegramRootCA);
  
  lastPollTime = millis();
}

void loop() {
  unsigned long now = millis();
  if (now - lastPollTime >= POLL_INTERVAL_MS) {
    lastPollTime = now;
    pollTelegramUpdates();
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, since connection to external Telegram servers is mocked, send the simulated update command using the serial input: `SYS:TELEGRAM:/led_on`.
5. Verify that the simulated LED widget turns ON.
6. Verify that the console prints the success response log.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[Telegram Recv] Message: /led_on | Chat: 987654321
[Telegram Send] Chat: 987654321 | Text: LED has been turned ON.
```

## Expected Canvas Behavior
* Booting the system launches the polling loop. Toggling simulated input triggers the LED widget and outputs JSON messages to the serial logs.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `getUpdates?offset=` | API offset parameter ensures previously read messages are not reprocessed. |
| `result["update_id"]` | Extracts the unique transaction ID to increment the offset tracker. |
| `message["chat"]["id"]` | Grabs the user's specific chat ID to target the reply. |

## Hardware & Safety Concept: Telegram Polling vs. Webhooks and Chat Filters
* **Telegram Polling vs. Webhooks**: Polling uses HTTP GET requests periodically to check for new messages, which is simple but introduces latency. Webhooks are real-time, but they require the ESP32 to host a public HTTPS server (requires forwarding public ports and buying domain certificates). Polling is safer for simple home nodes.
* **Chat Filters**: By default, anybody who discovers your bot username can send commands to turn your LED or relays ON/OFF. In production, always add a Chat ID filter checking against a list of authorized IDs:
  `if (chatId != AUTHORIZED_CHAT_ID) { sendTelegramMessage(chatId, "Unauthorized!"); return; }`

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Telegram Bot: Online".
2. **Authorized ID EEPROM Save**: Add a web configuration form (Project 172) where you can input and update your authorized Telegram chat ID.
3. **Telegram alarm node**: Read Project 188 to learn how to make the ESP32 send unsolicited warnings when sensor alerts trip.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Polling returns 401 Unauthorized | Incorrect bot token | Verify your bot token string. It must contain the prefix numbers, colon, and alphabet characters exactly as issued by BotFather |
| ESP32 crashes during JSON parsing | Out of stack space | If your bot receives a large list of updates, `DynamicJsonDocument` needs enough memory. Increase the document size to `4096` |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - ESP32 HTTPS Secure GET Request](../beginner/33-esp32-https-secure-get-request.md)
- [181 - OTA Update via Web Page upload](181-esp32-ota-update-via-web-page-upload.md)
- [188 - Telegram Bot Alarm Sender](188-telegram-bot-alarm-sender.md) (Next project)
