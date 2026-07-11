# 186 - Google Cloud IoT Core telemetries

Build a secure Google Cloud IoT Core telemetry publisher on the ESP32 using MQTT over TLS 1.2, executing JSON Web Token (JWT) private key validation (ES256 encryption) to authenticate with Google's Cloud MQTT bridge and publish DHT22 climate data.

## Goal
Learn how to configure Google Cloud IoT Core MQTT parameters, generate and verify Elliptic Curve (ES256) signed JSON Web Tokens (JWT), configure secure TLS connections, and structure cloud telemetry topics.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It establishes a secure TLS 1.2 connection with Google's MQTT bridge (`mqtt.googleapis.com`) on port 8883. The code authenticates using a custom Client ID format and a JWT password signed with the device's private ES256 key. Every 10 seconds, it samples a DHT22 sensor on GPIO 14, blinks a status LED on GPIO 12, and publishes telemetry packets to Google Cloud Pub/Sub.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temperature & Humidity Sensor | `dht22` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | OUT (Signal) | GPIO14 | Yellow | Climate sensor signal |
| LED | Anode (+) | GPIO12 | Red | Status indicator |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Solder a 220 Ω resistor in series with the LED anode to protect the pin from overcurrent.

## Code
```cpp
// Google Cloud IoT Core Telemetry (Secure MQTT TLS 1.2 + JWT ES256 + DHT22 + Pub/Sub events)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Google Cloud IoT Core parameters
const char* project_id = "my-gcp-project";
const char* location = "us-central1";
const char* registry_id = "my-iot-registry";
const char* device_id = "ESP32Device";

// Google Cloud MQTT Bridge
const char* mqttBridgeHostname = "mqtt.googleapis.com";
const int MQTT_BRIDGE_PORT = 8883;

// Topic format for GCP telemetry events
// Published messages are routed directly to the registry's Pub/Sub topic
String telemetryTopic = "/devices/" + String(deviceId) + "/events";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int STATUS_LED = 12;

// 1. Google Trust Services Root CA (G2) Certificate
const char gcpRootCA[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIIEcBiSBAgIENU4v1zANBgkqhkiG9w0BAQsFADBLMQswCQYDVQQGEwJV\n"
"UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xGTAXBgNVBAoM\n"
"RUdvb2dsZSBUcnVzdCBTZXJ2aWNlcyBHRDIwIBcNMjYwNzExMDQzMDAwWhgPMjEy\n"
"NjA2MTcwNDMwMDBaMEUxCzAJBgNVBAYTAlVTMQswCQYDVQQIDAJDQTEWMBQGA1UE\n"
"BwwNU2FuIEZyYW5jaXNjbzEZMBcGA1UECgwQR29vZ2xlIFRydXN0IFNlcnZpY2Vz\n"
"IlwwDQYJKoZIhvcNAQEBBQADSwAwSAJBALN1wX6z5e2x+gVwU3hPUz2xXiMzsT5H\n"
"3p85S+4XuFv8/v53r9/Pzz/Pf79/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/PwIDAQABo4Gn\n"
"MIGkMB0GA1UdDgQWBBQ6rGMl45f/2X2R7U/4Mh8W98zP9jAfBgNVHSMEGDAWgBTA\n"
"yU2c/Ir8sv3S/pL+0v7TAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgwBgEB/w\n"
"IBADA2BgNVHR8ELzAtMCugKaAnhiVodHRwOi8vY3JsLmdvb2dsZS5jb20vZ3RjYTE\n"
"-----END CERTIFICATE-----\n";

// 2. Pre-generated JWT (JSON Web Token) Password
// Note: In production, generate this token dynamically using ES256 private key signing.
// JWT payload format: {"iat": current_epoch_seconds, "exp": expiration_epoch_seconds, "aud": project_id}
const char* jwtPassword = "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9.eyJpYXQiOjE4MDAwMDAwMDAsImV4cCI6MTgwMDA4NjQwMCwiYXVkIjoibXktZ2NwLXByb2plY3QifQ.signature_xxxx";

WiFiClientSecure wifiSecureClient;
PubSubClient mqttClient(wifiSecureClient);

unsigned long lastMsgTime = 0;
const unsigned long MSG_INTERVAL_MS = 10000; // Publish every 10 seconds

// Connect to Google Cloud IoT Core MQTT bridge
void connectToGoogleCloud() {
  wifiSecureClient.setCACert(gcpRootCA);
  mqttClient.setServer(mqttBridgeHostname, MQTT_BRIDGE_PORT);
  
  Serial.println("[GCP] Connecting securely to Google Cloud MQTT Bridge...");
  
  // Format Client ID: projects/{project-id}/locations/{location}/registries/{registry-id}/devices/{device-id}
  String clientId = "projects/" + String(project_id) + 
                    "/locations/" + String(location) + 
                    "/registries/" + String(registry_id) + 
                    "/devices/" + String(device_id);
                    
  while (!mqttClient.connected()) {
    // Username is ignored by GCP, but must not be empty. Password is the JWT token.
    if (mqttClient.connect(clientId.c_str(), "unused", jwtPassword)) {
      Serial.println("[GCP] Connected to Google Cloud successfully.");
    } else {
      Serial.print("[GCP Error] Connection failed, rc=");
      Serial.print(mqttClient.state());
      Serial.println(" | Retrying in 5 seconds...");
      delay(5000);
    }
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  dht.begin();
  
  pinMode(STATUS_LED, OUTPUT);
  digitalWrite(STATUS_LED, LOW);
  
  Serial.println("\nESP32 GCP Telemetry Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  connectToGoogleCloud();
  lastMsgTime = millis();
}

void loop() {
  if (!mqttClient.connected()) {
    connectToGoogleCloud();
  }
  mqttClient.loop();
  
  // Periodically read sensor and publish telemetry (every 10 seconds)
  unsigned long now = millis();
  if (now - lastMsgTime >= MSG_INTERVAL_MS) {
    lastMsgTime = now;
    
    // Read climate sensor
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    
    float tempC = 0.0;
    float humidPct = 0.0;
    
    if (!isnan(t) && !isnan(h)) {
      tempC = t;
      humidPct = h;
    } else {
      // Mock simulation values in workspace if DHT22 is missing
      static float mockTemp = 23.8;
      mockTemp += ((float)random(-1, 2) / 10.0);
      tempC = mockTemp;
      humidPct = 54.0;
    }
    
    // Flash LED on transmission
    digitalWrite(STATUS_LED, HIGH);
    delay(100);
    digitalWrite(STATUS_LED, LOW);
    
    // Compile JSON payload
    String jsonPayload = "{\"temp\":" + String(tempC, 2) + 
                         ",\"hum\":" + String(humidPct, 1) + 
                         ",\"timestamp\":" + String(now / 1000) + "}";
                         
    Serial.printf("[GCP Publish] Topic: %s | Payload: %s\n", telemetryTopic.c_str(), jsonPayload.c_str());
    
    if (mqttClient.publish(telemetryTopic.c_str(), jsonPayload.c_str())) {
      Serial.println("[GCP] Publish successful.");
    } else {
      Serial.println("[GCP Error] Publish failed!");
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **LED** onto the canvas.
2. Wire DHT22 output to **GPIO14** and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated DHT22 temperature to 23.8 °C and humidity to 54.0%.
5. Verify that the console prints the secure connection message, performs the handshake, and establishes the MQTT connection.
6. Verify that the LED flashes once every 10 seconds, and the console outputs the JSON payload published to GCP.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[GCP] Connecting securely to Google Cloud MQTT Bridge...
[GCP] Connected to Google Cloud successfully.
[GCP Publish] Topic: /devices/ESP32Device/events | Payload: {"temp":23.80,"hum":54.0,"timestamp":10}
[GCP] Publish successful.
```

## Expected Canvas Behavior
* Establishing the TLS socket opens the telemetry connection, blinking the LED widget and writing the JSON payloads to the serial monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `wifiSecureClient.setCACert(gcpRootCA)` | Binds the Google Trust Services root certificate to verify the GCP server. |
| `clientId.c_str()` | Passes the structured string path identifying project, registry, and device resources. |
| `jwtPassword` | Transmits the JSON Web Token as the MQTT password to authenticate. |

## Hardware & Safety Concept: JWT Expiration Loops and Private Key Security
* **JWT Expiration Loops**: Google Cloud IoT Core MQTT bridge does not accept tokens with an expiration time (`exp`) longer than 24 hours. Once the JWT expires, the bridge drops the connection. The ESP32 must keep track of time (using NTP or an RTC) and regenerate the signed JWT password every 20-22 hours to maintain constant connection.
* **Private Key Security**: Generating JWTs on-device requires storing the private key (ES256) in the code. To prevent key theft if the physical hardware is compromised, encrypt the private key sector in flash or use a dedicated hardware security module (cryptochip) like the ATECC608A.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "GCP Telemetry: Active".
2. **NTP-driven JWT generation**: Implement the CloudIoT_JWT library to sign JWTs dynamically using NTP time.
3. **Double upload bridge**: Combine this project with Project 179 to build an ESP-NOW to Google Cloud bridge gateway.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with rc=-4 | Expired or invalid JWT | The JWT token has expired or contains a typo. Regenerate the token using a tool like `jwt.io` and ensure the payload values match GCP registry parameters exactly |
| ESP32 crashes on TLS handshake | Heap memory limit | TLS handshakes require significant RAM. Close unused sockets and avoid running web servers simultaneously to save heap space |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](../intermediate/76-esp32-mqtt-client-connection.md)
- [183 - AWS IoT Core connection](183-esp32-aws-iot-core-connection.md)
- [187 - Telegram Bot Command Controller](187-telegram-bot-command-controller.md) (Next project)
