# 82 - MQTT Subscribe Buzzer melody player

Configure the ESP32 as an MQTT subscriber node, register a callback handler, subscribe to the topic `esp32/control/buzzer`, and play distinct pre-programmed musical melody patterns on a passive buzzer connected to GPIO 15 based on text commands ("ALERT", "CHIME", "ERROR").

## Goal
Learn how to subscribe to MQTT topics, parse command payloads, generate multiple audio frequencies using the `tone()` API, and construct melody patterns.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A passive buzzer is on GPIO 15. The ESP32 connects to the HiveMQ broker and subscribes to `esp32/control/buzzer`. Publishing `ALERT` triggers a high-pitched pulsing alarm, publishing `CHIME` plays a rising three-tone chime, and publishing `ERROR` sounds a low error tone.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Passive Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | VCC (+) | GPIO15 | Blue | Audio signal channel |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the passive buzzer directly to GPIO 15.

## Code
```cpp
// MQTT Subscribe Buzzer melody player (Audio warning receiver node)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Configure MQTT Topics
const char* commandTopic = "esp32/control/buzzer";

const int BUZZER_PIN = 15;

WiFiClient espClient;
PubSubClient client(espClient);

// Play a high-pitched siren melody
void playAlertMelody() {
  for (int i = 0; i < 3; i++) {
    tone(BUZZER_PIN, 2000); // 2000 Hz
    delay(150);
    noTone(BUZZER_PIN);
    delay(100);
  }
}

// Play a rising three-note musical chime
void playChimeMelody() {
  // C5 (523 Hz), E5 (659 Hz), G5 (784 Hz)
  int notes[] = {523, 659, 784};
  
  for (int i = 0; i < 3; i++) {
    tone(BUZZER_PIN, notes[i]);
    delay(150);
    noTone(BUZZER_PIN);
    delay(50);
  }
}

// Play a low error buzz
void playErrorMelody() {
  tone(BUZZER_PIN, 220); // 220 Hz
  delay(400);
  noTone(BUZZER_PIN);
}

// MQTT Message Callback Handler
void callback(char* topic, byte* payload, unsigned int length) {
  // Convert payload array to string
  String message = "";
  for (int i = 0; i < length; i++) {
    message += (char)payload[i];
  }
  message.trim();
  
  Serial.printf("[MQTT Sub] Audio Command arrived -> Topic: %s | Payload: %s\n", topic, message.c_str());
  
  // Verify command topic
  if (String(topic) == commandTopic) {
    if (message == "ALERT") {
      Serial.println("[Action] Sounding ALERT siren...");
      playAlertMelody();
    } 
    else if (message == "CHIME") {
      Serial.println("[Action] Sounding CHIME melody...");
      playChimeMelody();
    } 
    else if (message == "ERROR") {
      Serial.println("[Action] Sounding ERROR tone...");
      playErrorMelody();
    } 
    else {
      Serial.println("[Error] Unknown command payload!");
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
  
  // In ESP32 Arduino Core, tone() pin should be set as output
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("\nESP32 MQTT Audio Subscriber Starting...");
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
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Wire Buzzer to **GPIO15**.
3. Paste the code and click **Run**.
4. In simulation, publish the command `ALERT`, `CHIME`, or `ERROR` to the topic `esp32/control/buzzer`. Watch the buzzer sound.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Audio Subscriber Starting...
....
WiFi Connected.
Attempting MQTT connection...connected.
Subscribed to topic: esp32/control/buzzer
[MQTT Sub] Audio Command arrived -> Topic: esp32/control/buzzer | Payload: CHIME
[Action] Sounding CHIME melody...
[MQTT Sub] Audio Command arrived -> Topic: esp32/control/buzzer | Payload: ALERT
[Action] Sounding ALERT siren...
```

## Expected Canvas Behavior
* Publishing `ALERT`, `CHIME`, or `ERROR` to the topic causes the buzzer widget to pulse green.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `tone(BUZZER_PIN, ...)` | Generates a square wave of the specified frequency on the buzzer pin. |
| `noTone(BUZZER_PIN)` | Stops wave generation, silencing the pin output. |
| `playChimeMelody()` | Sequential list of notes played to generate a musical chime. |
| `client.loop()` | Processes background frames to parse subscription events. |

## Hardware & Safety Concept: Non-Blocking Audio Operations in Callbacks
Melody patterns require turning the pin HIGH and LOW with precise delays (e.g. `delay(150)`). While this is blocking, keeping these durations short (under 500 ms) ensures the MQTT client remains responsive to new requests. For longer alarms (like a continuous siren), manage the alarm state using non-blocking timers (Project 29).

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a checklist showing the last command received.
2. **Frequency Selector**: Add a topic `esp32/control/buzzer/freq` and set the buzzer frequency directly.
3. **Emergency Auto-Timeout**: Add code to silence the buzzer automatically after 5 seconds of continuous playback to prevent coil overheating.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm that the passive buzzer is connected to GPIO 15 |
| Sound is soft or distorted | Resistor in series | Ensure the passive buzzer is powered directly from the output pin (verify maximum current limits) |
| Commands are ignored | Callback not registered | Ensure `client.setCallback(callback)` is called in setup |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [45 - ESP32 Web-based Buzzer chime trigger](../beginner/45-esp32-web-based-buzzer-chime-trigger.md)
- [55 - ESP32 Web Page Buzzer frequency selector](55-esp32-web-page-buzzer-frequency-selector.md)
- [81 - ESP32 MQTT Subscribe Relay controller](81-esp32-mqtt-subscribe-relay-controller.md)
