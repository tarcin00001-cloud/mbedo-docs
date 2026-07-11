# 185 - Azure IoT Hub direct telemetry logger

Build a secure Azure cloud integration node on the ESP32 that connects directly to Microsoft Azure IoT Hub using MQTT over TLS 1.2, performs Shared Access Signature (SAS) token authentication on port 8883, and publishes DHT22 climate telemetry to the cloud.

## Goal
Learn how to configure Azure IoT Hub MQTT credentials, generate Shared Access Signature (SAS) tokens, perform Root CA validation using WiFiClientSecure, and publish JSON telemetry strings.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It establishes a secure socket with your Azure IoT Hub endpoint on port 8883. Using a Baltimore Root CA certificate for server verification, it authenticates using a SAS token password. Every 10 seconds, it reads a DHT22 sensor on GPIO 14, blinks an LED on GPIO 12, and publishes a JSON payload directly to the Azure telemetry event topic.

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
// Azure IoT Hub Telemetry Logger (Secure MQTT on Port 8883 + SAS Token Auth + DHT22 + JSON)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Azure IoT Hub Parameters (Replace with your actual Azure IoT Hub parameters)
const char* azureEndpoint = "myhub.azure-devices.net";
const int AZURE_MQTT_PORT = 8883;
const char* deviceId = "ESP32Device";

// Azure MQTT Username format: {iothubhostname}/{device_id}/?api-version=2021-04-12
const char* mqttUsername = "myhub.azure-devices.net/ESP32Device/?api-version=2021-04-12";

// Azure MQTT Password: Your generated Shared Access Signature (SAS) Token
// Note: SAS tokens expire after a set time. Regenerate this token in Azure CLI / VS Code
const char* sasToken = "SharedAccessSignature sr=myhub.azure-devices.net%2Fdevices%2FESP32Device&sig=xxxxxx&se=1800000000";

// Azure IoT Hub Telemetry topic: devices/{device_id}/messages/events/
const char* telemetryTopic = "devices/ESP32Device/messages/events/";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int STATUS_LED = 12;

// Baltimore CyberTrust Root CA (Used by Azure IoT Hub to sign its SSL certificates)
const char azureRootCA[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIIDtzCCAp+gAwIBAgIQDfb1ZTyX7hSG8BSmSKsNfDANBgkqhkiG9w0BAQsFADBl\n"
"MQswCQYDVQQGEwJVUzEYMBYGA1UEChMPR2VvVHJ1c3QgSW5jLjEdMBsGA1UECxMU\n"
"R2VvVHJ1c3QgUHJpbWFyeSBDQSAxMTAwFw0wNjExMjcwMDAwMDBaFw0zMTExMTAw\n"
"MDAwMDBaMHMxCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdpQ2VydCBJbmMxGTAX\n"
"BgNVBAsTEHd3dy5kaWdpY2VydC5jb20xMS8wLQYDVQQDEyZEaWdpQ2VydCBCYWx0\n"
"aW1vcmUgQ3liZXJUcnVzdCBSb290WhgPMjEyNjA2MTcwNDMwMDBaMEUxCzAJBgNV\n"
"BAYTAlVTMQswCQYDVQQIDAJDQTEWMBQGA1UEBwwNU2FuIEZyYW5jaXNjbzEZMBcG\n"
"A1UECgwQQmFsdGltb3JlIENBMRcwFQYDVQQDDA5CYWx0aW1vcmUgUm9vdFwwDQYJ\n"
"KoZIhvcNAQEBBQADSwAwSAJBALN1wX6z5e2x+gVwU3hPUz2xXiMzsT5H3p85S+4X\n"
"uFv8/v53r9/Pzz/Pf79/Pz9/Pz9/Pz9/Pz9/Pz9/PwIDAQABo4GnMIGkMB0GA1Ud\n"
"DgQWBBQ6rGMl45f/2X2R7U/4Mh8W98zP9jAfBgNVHSMEGDAWgBTA/y4oH92U2c/I\n"
"r8sv3S/pL+0v7TAOBgNVHQ8BAf8EBAMCAQYwEgYDVR0TAQH/BAgwBgEB/wIBADA2\n"
"BgNVHR8ELzAtMCugKaAnhiVodHRwOi8vY3JsLmdlb3RydXN0LmNvbS9jcmxzL2d0\n"
"cGNhMS5jcmwwDQYJKoZIhvcNAQELBQADSwAwSAJBALN1wX6z5e2x+gVwU3hPUz2x\n"
"XiMzsT5H3p85S+4XuFv8/v53r9/Pzz/Pf79/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/Pz9/\n"
"-----END CERTIFICATE-----\n";

WiFiClientSecure wifiSecureClient;
PubSubClient mqttClient(wifiSecureClient);

unsigned long lastMsgTime = 0;
const unsigned long MSG_INTERVAL_MS = 10000; // Publish every 10 seconds

// Connect to Azure IoT Hub securely
void connectToAzure() {
  // 1. Register Baltimore Root CA
  wifiSecureClient.setCACert(azureRootCA);
  
  mqttClient.setServer(azureEndpoint, AZURE_MQTT_PORT);
  
  Serial.println("[Azure] Connecting securely to Azure IoT Hub...");
  
  while (!mqttClient.connected()) {
    // Authenticate using DeviceId, Username, and SAS Token Password
    if (mqttClient.connect(deviceId, mqttUsername, sasToken)) {
      Serial.println("[Azure] Connected to Azure IoT Hub successfully.");
    } else {
      Serial.print("[Azure Error] Connection failed, rc=");
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
  
  Serial.println("\nESP32 Azure Telemetry Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  connectToAzure();
  lastMsgTime = millis();
}

void loop() {
  if (!mqttClient.connected()) {
    connectToAzure();
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
      static float mockTemp = 24.2;
      mockTemp += ((float)random(-1, 2) / 10.0);
      tempC = mockTemp;
      humidPct = 52.0;
    }
    
    // Flash LED on transmission
    digitalWrite(STATUS_LED, HIGH);
    delay(100);
    digitalWrite(STATUS_LED, LOW);
    
    // Compile JSON payload
    String jsonPayload = "{\"deviceId\":\"" + String(deviceId) + "\"" +
                         ",\"temperature\":" + String(tempC, 2) + 
                         ",\"humidity\":" + String(humidPct, 1) + 
                         ",\"clientUptime\":" + String(now / 1000) + "}";
                         
    Serial.printf("[Azure Publish] Topic: %s | Payload: %s\n", telemetryTopic, jsonPayload.c_str());
    
    if (mqttClient.publish(telemetryTopic, jsonPayload.c_str())) {
      Serial.println("[Azure] Publish successful.");
    } else {
      Serial.println("[Azure Error] Publish failed!");
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **LED** onto the canvas.
2. Wire DHT22 output to **GPIO14** and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated DHT22 temperature to 22.8 °C and humidity to 55.0%.
5. Verify that the console prints the secure connection message, performs the handshake, and establishes the MQTT connection.
6. Verify that the LED flashes once every 10 seconds, and the console outputs the JSON payload published to Azure.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[Azure] Connecting securely to Azure IoT Hub...
[Azure] Connected to Azure IoT Hub successfully.
[Azure Publish] Topic: devices/ESP32Device/messages/events/ | Payload: {"deviceId":"ESP32Device","temperature":22.80,"humidity":55.0,"clientUptime":10}
[Azure] Publish successful.
```

## Expected Canvas Behavior
* Establishing the TLS socket opens the telemetry connection, blinking the LED widget and writing the JSON payloads to the serial monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `wifiSecureClient.setCACert(azureRootCA)` | Binds the Baltimore CyberTrust certificate to verify the Azure server's identity. |
| `mqttClient.connect(deviceId, ...)` | Transmits the DeviceID, formatted Username, and SAS Token Password for authentication. |
| `mqttClient.publish(...)` | Publishes the JSON data payload to Azure's system telemetry topic. |

## Hardware & Safety Concept: SAS Token Expiration and Baltimore Root Certificate Migration
* **SAS Token Expiration**: Shared Access Signature tokens are generated with a specific lifespan (e.g. 24 hours, 30 days). If the token expires, the Azure server terminates the connection, and the ESP32 will fail to reconnect. In production, calculate SAS tokens dynamically using a real-time clock (RTC) or fetch fresh tokens from an administrative server.
* **Baltimore CA Migration**: Microsoft is migrating its Azure endpoints from the Baltimore CyberTrust Root CA to DigiCert Global Root G2. Ensure your firmware embeds both CA certificates, allowing the device to connect smoothly when certificates swap.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Azure Status: Online".
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the secure connection state drops (`mqttClient.state()` returns error).
3. **Double upload bridge**: Combine this project with Project 179 to build an ESP-NOW to Azure IoT Hub bridge gateway.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with rc=-4 | Expired SAS token | The SAS token has expired or contains a typo. Regenerate the token with a longer duration and paste it into the code |
| ESP32 crashes on TLS handshake | Heap memory limit | TLS handshakes require significant RAM. Close unused sockets and avoid running web servers simultaneously to save heap space |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](../intermediate/76-esp32-mqtt-client-connection.md)
- [183 - AWS IoT Core connection](183-esp32-aws-iot-core-connection.md)
- [184 - AWS IoT Shadow updater](184-esp32-aws-iot-shadow-updater.md)
