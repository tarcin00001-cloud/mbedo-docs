# 77 - MQTT Broker authentication (User/Password credentials)

Configure a secured MQTT connection on the ESP32 that passes username and password authorization parameters during the broker handshake, and monitors the connection status on the Serial Monitor.

## Goal
Learn how to use MQTT authorization parameters, secure broker connections, generate unique Client IDs, and diagnose connection errors.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It establishes a connection to an MQTT broker, passing a username (`iot_node_1`) and password (`securepass123`) in the connection frame. It prints the status of the connection handshake to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// MQTT Broker authentication (Secured Broker Handshake)
#include <WiFi.h>
#include <PubSubClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Secured MQTT Broker Configuration
const char* mqttServer = "broker.hivemq.com";
const int mqttPort = 1883;

// Authorization Parameters
const char* mqttUsername = "iot_node_1";
const char* mqttPassword = "securepass123";

WiFiClient espClient;
PubSubClient client(espClient);

void callback(char* topic, byte* payload, unsigned int length) {
  // Callback handler details can go here
}

// Reconnection loop enforcing authentication
void reconnect() {
  while (!client.connected()) {
    Serial.print("Attempting secured MQTT connection...");
    
    // Generate unique Client ID
    String clientId = "ESP32Client-" + String(random(0xffff), HEX);
    
    // 1. Initiate connection passing authorization parameters
    // Format: client.connect(clientId, username, password)
    if (client.connect(clientId.c_str(), mqttUsername, mqttPassword)) {
      Serial.println("connected and authorized successfully.");
      
      // Publish initial state
      client.publish("esp32/security/status", "AUTHORIZED");
    } else {
      // 2. Evaluate error code
      // Common codes: 4 (Bad Credentials), 5 (Unauthorized)
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
  
  Serial.println("\nESP32 Secured MQTT Client Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Configure broker connection
  client.setServer(mqttServer, mqttPort);
  client.setCallback(callback);
}

void loop() {
  if (!client.connected()) {
    reconnect();
  }
  
  client.loop();
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the authorized connection logs.

## Expected Output
Serial Monitor:
```
ESP32 Secured MQTT Client Starting...
....
WiFi Connected.
Attempting secured MQTT connection...connected and authorized successfully.
```

## Expected Canvas Behavior
* The Serial Monitor logs connection and authorization states.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `client.connect(..., username, password)` | Transmits credentials in the MQTT connection frame payload. |
| `client.state()` | Returns the connection state or error code if the handshake fails. |
| `client.loop()` | Processes background keepalive frames. |

## Hardware & Safety Concept: MQTT Authentication and Transport Encryption
* **Authentication**: Enforcing username and password validation prevents unauthorized devices from joining the broker network and intercepting or sending control commands.
* **TLS Encryption**: Basic MQTT credentials are sent in plain text. For production security, combine credentials with TLS encryption using `WiFiClientSecure` on port 8883 to encrypt the payloads.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a lock icon showing if the connection is authorized.
2. **Invalid Credentials test**: Modify the password to an incorrect string and observe the error code returned in the Serial Monitor.
3. **Buzzer alert on fail**: Sound a warning chirp if authorization fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with rc = 4 | Bad credentials | Verify that the username and password match the broker's configuration |
| Connection fails with rc = 5 | Unauthorized access | Confirm that the client has permissions to connect to the broker |
| Sockets drop frequently | Keepalive ping missing | Ensure `client.loop()` is called continuously in the loop function |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](76-esp32-mqtt-client-connection.md)
- [78 - ESP32 MQTT Topic publisher (Sensor telemetry stream)](78-esp32-mqtt-topic-publisher-sensor-telemetry-stream.md) (Next project)
- [79 - ESP32 MQTT Topic subscriber (Remote pin control command)](79-esp32-mqtt-topic-subscriber-remote-pin-control-command.md)
@
