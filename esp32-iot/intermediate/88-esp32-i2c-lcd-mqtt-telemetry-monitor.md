# 88 - I2C LCD MQTT telemetry monitor (local prints)

Build an MQTT monitoring station on the ESP32 that connects to a local WiFi network, subscribes to climate telemetry topics (`esp32/telemetry/temp` and `esp32/telemetry/humidity`), and prints the incoming real-time climate values on a 16x2 I2C LCD.

## Goal
Learn how to subscribe to multiple MQTT topics, process multiple payload values, manage persistent display refreshes on I2C LCDs, and handle connection drops.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). The ESP32 connects to the public HiveMQ broker and subscribes to two topics: `esp32/telemetry/temp` and `esp32/telemetry/humidity`. When other devices publish values to these topics, the ESP32 captures them and displays them on the LCD in real time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the LCD from the 5V Vin rail.

## Code
```cpp
// I2C LCD MQTT telemetry monitor (Climate Subscriber Display Node)
#include <WiFi.h>
#include <PubSubClient.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics to subscribe
const char* tempTopic = "esp32/telemetry/temp";
const char* humTopic = "esp32/telemetry/humidity";

WiFiClient espClient;
PubSubClient client(espClient);

// Global variables to cache telemetry values
String lastTemp = "--.-";
String lastHum = "--.-";

void updateLCD() {
  lcd.clear();
  
  // Row 0: Temperature Display
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(lastTemp);
  lcd.print(" C");
  
  // Row 1: Humidity Display
  lcd.setCursor(0, 1);
  lcd.print("Hum:  ");
  lcd.print(lastHum);
  lcd.print(" %");
}

// MQTT Message Callback Handler
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Sub] Telemetry arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Identify which topic value updated
  if (String(topic) == tempTopic) {
    lastTemp = message;
    updateLCD();
  } 
  else if (String(topic) == humTopic) {
    lastHum = message;
    updateLCD();
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      
      // Subscribe to both topics on reconnection
      client.subscribe(tempTopic);
      client.subscribe(humTopic);
      
      Serial.println("Subscribed to telemetry topics.");
      updateLCD();
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" | Retrying in 5 seconds...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("MQTT Monitor");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  Serial.println("\nESP32 MQTT LCD Monitor Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback); // Register callback function
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  
  // Yield background processing to handle subscription events
  client.loop();
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. In simulation, publish values (e.g. `24.5` and `45.0`) to the topics `esp32/telemetry/temp` and `esp32/telemetry/humidity`. Watch the LCD screen update.

## Expected Output
Serial Monitor:
```
ESP32 MQTT LCD Monitor Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to telemetry topics.
[MQTT Sub] Telemetry arrived -> Topic: esp32/telemetry/temp | Payload: 24.5
[MQTT Sub] Telemetry arrived -> Topic: esp32/telemetry/humidity | Payload: 45.0
```

LCD Display:
```
Temp: 24.5 C
Hum:  45.0 %
```

## Expected Canvas Behavior
* Publishing data values to the subscribed topics updates the text displayed on the LCD widget.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.subscribe(...)` | Registers the subscription filter on the broker for both topics. |
| `callback(...)` | Executed when a message matching either subscription arrives. |
| `updateLCD()` | Clears and updates the LCD screen with the latest cached telemetry values. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Cache Variables and Asynchronous Display Refreshing
Writing data to I2C displays (like LCDs) involves transmitting bytes over serial buses. If a device writes to the LCD continuously in the loop, it blocks the CPU, increasing latency. By caching the telemetry values in global variables (`lastTemp`, `lastHum`) and only refreshing the LCD when a new value arrives inside the callback, the system remains responsive.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a graph of temperature changes.
2. **Out of bounds warning**: Sound an alarm on a buzzer (GPIO 15) if the temperature exceeds a safety threshold.
3. **SPIFFS Logging**: Write code to log incoming telemetry values to a file in SPIFFS (Project 62).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |
| The ESP32 does not receive commands | `client.loop()` missing | Verify that `client.loop()` is called continuously in the loop function |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [78 - ESP32 MQTT Topic publisher (Sensor telemetry stream)](78-esp32-mqtt-topic-publisher-sensor-telemetry-stream.md)
- [79 - ESP32 MQTT Topic subscriber (Remote pin control command)](79-esp32-mqtt-topic-subscriber-remote-pin-control-command.md)
