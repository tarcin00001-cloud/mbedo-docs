# 86 - MQTT JSON Payload subscriber (deserialize commands)

Configure the ESP32 as a multi-actuator MQTT subscriber node, subscribe to the command topic `esp32/control/json`, parse incoming structured JSON command payloads (e.g. `{"led":1,"servo":90}`) using the ArduinoJson library, actuate an LED on GPIO 13 and a servo on GPIO 12, and publish status updates.

## Goal
Learn how to subscribe to MQTT topics, deserialize incoming JSON payloads, extract multiple command keys, control different actuators, and handle parsing errors.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An LED is on GPIO 13, and a servo motor on GPIO 12. The ESP32 connects to the HiveMQ broker and subscribes to `esp32/control/json`. Publishing a JSON payload like `{"led":1,"servo":90}` turns the LED ON and positions the servo to 90 degrees. The ESP32 publishes a confirmation payload back to the broker.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled status LED |
| Servo Motor | PWM (Signal) | GPIO12 | Yellow | Controlled axis servo |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor. Power the servo from the 5V Vin rail.

## Code
```cpp
// MQTT JSON Payload subscriber (Deserialized Command Actuator Node)
#include <WiFi.h>
#include <PubSubClient.h>
#include <ESP32Servo.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics
const char* commandTopic = "esp32/control/json";
const char* feedbackTopic = "esp32/status/actuators";

const int LED_PIN = 13;
const int SERVO_PIN = 12;
Servo myServo;

WiFiClient espClient;
PubSubClient client(espClient);

// MQTT Message Callback Handler
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT JSON Sub] Command arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Verify command topic
  if (String(topic) == commandTopic) {
    // 1. Allocate StaticJsonDocument buffer (256 bytes is sufficient)
    StaticJsonDocument<256> doc;
    
    // 2. Deserialize the JSON string payload
    DeserializationError error = deserializeJson(doc, message);
    
    if (!error) {
      Serial.println("[JSON] Parsing successful!");
      
      // 3. Actuate LED if "led" key exists
      if (doc.containsKey("led")) {
        int ledCmd = doc["led"];
        if (ledCmd == 1) {
          digitalWrite(LED_PIN, HIGH);
          Serial.println("[Action] LED set to HIGH.");
        } else if (ledCmd == 0) {
          digitalWrite(LED_PIN, LOW);
          Serial.println("[Action] LED set to LOW.");
        }
      }
      
      // 4. Actuate Servo if "servo" key exists
      if (doc.containsKey("servo")) {
        int servoAngle = doc["servo"];
        if (servoAngle >= 0 && servoAngle <= 180) {
          myServo.write(servoAngle);
          Serial.printf("[Action] Servo positioned to %d degrees.\n", servoAngle);
        } else {
          Serial.printf("[Warning] Out of bounds servo angle: %d\n", servoAngle);
        }
      }
      
      // 5. Publish status confirmation feedback
      bool currentLED = (digitalRead(LED_PIN) == HIGH);
      int currentAngle = myServo.read();
      
      StaticJsonDocument<128> feedbackDoc;
      feedbackDoc["led"] = currentLED ? 1 : 0;
      feedbackDoc["servo"] = currentAngle;
      
      char feedbackBuffer[128];
      serializeJson(feedbackDoc, feedbackBuffer);
      client.publish(feedbackTopic, feedbackBuffer);
    } else {
      Serial.print("[JSON Error] Deserialization failed: ");
      Serial.println(error.c_str());
    }
  }
}

void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      
      // Subscribe to command channel on reconnection
      client.subscribe(commandTopic);
      Serial.printf("Subscribed to topic: %s\n", commandTopic);
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
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  myServo.attach(SERVO_PIN);
  myServo.write(90); // Center servo (90 degrees) on startup
  
  Serial.println("\nESP32 MQTT JSON Subscriber Starting...");
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
1. Drag **ESP32 DevKitC**, **LED**, and **Servo** onto the canvas.
2. Wire LED to **GPIO13** and Servo to **GPIO12**.
3. Paste the code and click **Run**.
4. In simulation, publish the command payload `{"led":1,"servo":45}` to the topic `esp32/control/json` using an MQTT client utility. Watch the LED light up and the servo rotate.

## Expected Output
Serial Monitor:
```
ESP32 MQTT JSON Subscriber Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/json
[MQTT JSON Sub] Command arrived -> Topic: ... | Payload: {"led":1,"servo":45}
[JSON] Parsing successful!
[Action] LED set to HIGH.
[Action] Servo positioned to 45 degrees.
```

## Expected Canvas Behavior
* Publishing the JSON command payload switches the Red LED widget ON and rotates the servo widget in real time.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `deserializeJson(doc, message)` | Deserializes the JSON string into the document memory buffer. |
| `doc.containsKey("led")` | Verifies if the target key is present before reading the value to prevent errors. |
| `myServo.write(servoAngle)` | Positions the servo motor shaft to the target angle. |
| `client.publish(feedbackTopic, ...)` | Publishes a status confirmation JSON package back to the broker. |

## Hardware & Safety Concept: Actuator Protection and Boundary Limits
When driving physical hardware from JSON control packets:
1. **Key Verification**: Use `doc.containsKey()` to verify if keys exist before reading them. Attempting to read a missing key can return 0 or default values, causing unintended movements.
2. **Boundary Checks**: Validate target parameters (such as restricting angles between 0° and 180°) before actuating.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing active actuator states.
2. **Add Buzzer Control**: Expand the JSON command parsing to handle a `buzzer` key (e.g. `{"buzzer":1}`) to beep a buzzer.
3. **Emergency Lockout**: Set a safety shutdown flag if a command payload fails to parse.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Deserialization error | Payload format wrong | Verify that the published payload is valid JSON (e.g. double quotes around keys) |
| Actuators do not move | Key names wrong | Verify that the published JSON keys match the code parameters (`led` and `servo`) |
| Servo jitter | Noise on signal line | Install a 100 µF capacitor across the servo power lines |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [38 - ESP32 HTTP JSON API Parser](../beginner/38-esp32-http-json-api-parser.md)
- [83 - ESP32 MQTT Subscribe Servo Position controller](83-esp32-mqtt-subscribe-servo-position-controller.md)
- [85 - ESP32 MQTT JSON Payload publisher](85-esp32-mqtt-json-payload-publisher-serialize-dht22-values.md)
