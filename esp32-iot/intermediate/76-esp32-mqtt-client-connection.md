# 76 - MQTT Client Connection (join broker SSID)

Configure the ESP32 as an MQTT client using the PubSubClient library, establish a connection to a public MQTT broker (`broker.hivemq.com`) on port 1883, and handle connection drops dynamically.

## Goal
Learn how to configure MQTT client setups, manage reconnection loops, register callback handlers, and monitor broker connection states.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and establishes a persistent TCP connection to the public MQTT broker `broker.hivemq.com`. It prints the status of the connection (Connected or Disconnected) to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// MQTT Client Connection (PubSubClient Broker Link)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Public MQTT Broker
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

WiFiClient espClient;
PubSubClient client(espClient);

// 1. MQTT Message Callback Handler
// Triggered automatically when the client receives a message on subscribed topics
void callback(char* topic, byte* payload, unsigned int length) {
  Serial.print("[MQTT Callback] Message arrived on topic: ");
  Serial.print(topic);
  Serial.print(" | Payload: ");
  
  for (int i = 0; i < length; i++) {
    Serial.print((char)payload[i]);
  }
  Serial.println();
}

// 2. MQTT Reconnection Loop
// Enforces a non-blocking loop to reconnect to the broker if connection drops
void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting MQTT connection...");
    
    // Generate a unique Client ID based on the ESP32 MAC address
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    // Attempt to connect
    if (client.connect(clientId.c_str())) {
      Serial.println("connected.");
      
      // Send a test message to confirm connection
      client.publish("esp32/status", "ONLINE");
    } else {
      Serial.print("failed, rc=");
      Serial.print(client.state());
      Serial.println(" | Trying again in 5 seconds...");
      
      // Wait 5 seconds before retrying
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("\nESP32 MQTT Client Setup Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // 3. Configure MQTT Client
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback); // Register callback function
}

void loop() {
  // 4. Ensure connection is active
  if (!client.connected()) {
    reconnect();
  }
  
  // 5. Yield background processing to the MQTT engine
  client.loop();
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the MQTT connection completes.

## Expected Output
Serial Monitor:
```
ESP32 MQTT Client Setup Starting...
....
WiFi Connected successfully.
Attempting MQTT connection...connected.
```

## Expected Canvas Behavior
* The Serial Monitor logs connection states and keeps the socket active.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `PubSubClient client(...)` | Instantiates an MQTT client object using the standard WiFi client socket. |
| `client.setServer(...)` | Configures the target MQTT broker address and port. |
| `client.connect(...)` | Generates a connection handshake frame, passing a unique Client ID. |
| `client.loop()` | Processes background frames to maintain keepalive pings. |

## Hardware & Safety Concept: MQTT Keepalive Pings and Unique Client IDs
* **Keepalive Pings**: MQTT connections are persistent. If a device goes silent, the broker can close the socket. `client.loop()` sends periodic ping frames (keepalives) in the background to confirm the link is active.
* **Unique Client IDs**: Brokers identify devices using Client IDs. If two devices connect with the same ID, the broker will disconnect the older socket, causing a connection loop. Always generate a unique ID (e.g. using MAC addresses).

## Try This! (Challenges)
1. **OLED Connection Dashboard**: Add an OLED screen (Project 60) and display the current MQTT connection state.
2. **Interactive Status switch**: Add a button on GPIO 12 to send an emergency trigger message on topic `esp32/alert`.
3. **Buzzer alert on join**: Sound a quick beep on a buzzer (GPIO 15) when the connection completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails (rc = -2) | Broker unreachable / DNS failed | Verify the broker hostname is correct and the gateway has internet access |
| Connection loops constantly | Client ID conflict | Ensure you generate a unique Client ID for each connection |
| Client disconnects instantly | Keepalive ping missing | Ensure `client.loop()` is called continuously in the loop function without blocking delays |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [16 - ESP32 TCP Client connection test](../beginner/16-esp32-tcp-client-connection-test.md)
- [77 - ESP32 MQTT Broker authentication (User/Password credentials)](77-esp32-mqtt-broker-authentication-user-password-credentials.md) (Next project)
- [78 - ESP32 MQTT Topic publisher (Sensor telemetry stream)](78-esp32-mqtt-topic-publisher-sensor-telemetry-stream.md)
