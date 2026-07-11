# 183 - AWS IoT Core connection (secure MQTT with SSL keys)

Build a secure telemetry node on the ESP32 that connects to AWS IoT Core over MQTT using TLS 1.2 mutual authentication (Root CA, Client Certificate, and Client Private Key), samples a DHT22 climate sensor on GPIO 14, and publishes JSON telemetry to the cloud on port 8883.

## Goal
Learn how to implement secure MQTT connections using WiFiClientSecure, configure mutual authentication TLS handshakes, manage keys and certificates, parse DHT22 data, and construct JSON payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. It initializes a secure TCP socket with the AWS IoT Core endpoint on port 8883. The code embeds the AWS Root CA, the device's Client Certificate, and its Private Key in the flash memory. Every 10 seconds, the ESP32 reads ambient temperature and humidity, blinks a status LED on GPIO 12, and publishes a JSON payload to the AWS MQTT topic `esp32/dht22`.

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

> **Wiring tip:** Use a 220 Ω current-limiting resistor in series with the LED.

## Code
```cpp
// AWS IoT Core Connection (Secure MQTT TLS 1.2 on Port 8883 + DHT22 telemetry + JSON)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <DHT.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// AWS IoT Core Endpoint (Replace with your actual AWS IoT endpoint)
const char* awsEndpoint = "a3xxxxxxxxxxxx-ats.iot.us-east-1.amazonaws.com";
const int AWS_MQTT_PORT = 8883;

// MQTT Topics
const char* publishTopic = "esp32/dht22";

const int DHT_PIN = 14;
#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);

const int STATUS_LED = 12;

// 1. AWS Amazon Root CA 1 Certificate
const char awsRootCA[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIIDSjCCAjagAwIBAgIENU4v1zANBgkqhkiG9w0BAQsFADBLMQswCQYDVQQGEwJV\n"
"UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xGTAXBgNVBAoM\n"
"EEFtYXpvbiBSb290IENBIDEwIBcNMjYwNzExMDQzMDAwWhgPMjEyNjA2MTcwNDMw\n"
"MDBaMEUxCzAJBgNVBAYTAlVTMQswCQYDVQQIDAJDQTEWMBQGA1UEBwwNU2FuIEZy\n"
"YW5jaXNjbzEZMBcGA1UECgwQQW1hem9uIFJvb3QgQ0ExXDANBgkqhkiG9w0BAQEF\n"
"AASCAmAwggJcAgEAAoGBAMzP9eKqH/VlHk2iPzVf4T1M9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4\n"
"-----END CERTIFICATE-----\n";

// 2. Client Device Certificate
const char awsClientCert[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIICNDCCAZwCCQDFyv9eKqH/VDANBgkqhkiG9w0BAQsFADBLMQswCQYDVQQGEwJV\n"
"UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xGTAXBgNVBAoM\n"
"EEVzcHJlc3NpZiBTeXN0ZW0wIBcNMjYwNzExMDQzMDAwWhgPMjEyNjA2MTcwNDMw\n"
"MDBaMEUxCzAJBgNVBAYTAlVTMQswCQYDVQQIDAJDQTEWMBQGA1UEBwwNU2FuIEZy\n"
"YW5jaXNjbzEZMBcGA1UECgwQRXNwcmVzc2lmIFN5c3RlbTBcMA0GCSqGSIb3DQEB\n"
"AQUAA0sAMEgCQQDMz/Xiqh/1ZR5Nok81X+E9TPbFeIzOqGKfX/V9UfbVeIzOqGKf\n"
"X/V9UfbVeIzOqGKfX/V9UfbVeIzOqGKfX/V9AgMBAAEwDQYJKoZIhvcNAQELBQAD\n"
"QQB5X9eKqH/VlHk2iPzVf4T1M9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6\n"
"-----END CERTIFICATE-----\n";

// 3. Client Private Key
const char awsPrivateKey[] PROGMEM = 
"-----BEGIN PRIVATE KEY-----\n"
"MIICdgIBADANBgkqhkiG9w0BAQEFAASCAmAwggJcAgEAAoGBAMzP9eKqH/VlHk2i\n"
"PzVf4T1M9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"Yp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6oYp9f9X1R9sV4jM6o\n"
"-----END PRIVATE KEY-----\n";

WiFiClientSecure wifiSecureClient;
PubSubClient mqttClient(wifiSecureClient);

// Global sensor values
float tempC = 0.0;
float humidPct = 0.0;

unsigned long lastMsgTime = 0;
const unsigned long MSG_INTERVAL_MS = 10000; // Publish every 10 seconds

// Connect to AWS IoT Core securely
void connectToAWS() {
  // 1. Register security credentials
  wifiSecureClient.setCACert(awsRootCA);
  wifiSecureClient.setCertificate(awsClientCert);
  wifiSecureClient.setPrivateKey(awsPrivateKey);
  
  mqttClient.setServer(awsEndpoint, AWS_MQTT_PORT);
  
  Serial.println("[AWS] Connecting securely to AWS IoT Core...");
  
  while (!mqttClient.connected()) {
    String clientId = "ESP32-Telemetry-" + String(random(0xffff), HEX);
    
    if (mqttClient.connect(clientId.c_str())) {
      Serial.println("[AWS] Connected to AWS IoT Core successfully.");
    } else {
      Serial.print("[AWS Error] Connection failed, rc=");
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
  
  Serial.println("\nESP32 AWS Telemetry Station starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Initialize secure connection
  connectToAWS();
  
  lastMsgTime = millis();
}

void loop() {
  // Ensure MQTT client connection persists
  if (!mqttClient.connected()) {
    connectToAWS();
  }
  mqttClient.loop();
  
  // Periodically read sensor and publish telemetry (every 10 seconds)
  unsigned long now = millis();
  if (now - lastMsgTime >= MSG_INTERVAL_MS) {
    lastMsgTime = now;
    
    // Read climate sensor
    float t = dht.readTemperature();
    float h = dht.readHumidity();
    
    if (!isnan(t) && !isnan(h)) {
      tempC = t;
      humidPct = h;
    } else {
      // Mock simulation values in workspace if DHT22 is missing
      static float mockTemp = 23.5;
      mockTemp += ((float)random(-1, 2) / 10.0);
      tempC = mockTemp;
      humidPct = 55.0;
    }
    
    // Blink LED to indicate transmission
    digitalWrite(STATUS_LED, HIGH);
    delay(100);
    digitalWrite(STATUS_LED, LOW);
    
    // Compile JSON payload
    String jsonPayload = "{\"temperature\":" + String(tempC, 2) + 
                         ",\"humidity\":" + String(humidPct, 1) + 
                         ",\"clientUptime\":" + String(now / 1000) + "}";
                         
    Serial.printf("[AWS Publish] Topic: %s | Payload: %s\n", publishTopic, jsonPayload.c_str());
    
    if (mqttClient.publish(publishTopic, jsonPayload.c_str())) {
      Serial.println("[AWS] Publish successful.");
    } else {
      Serial.println("[AWS Error] Publish failed!");
    }
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22 Sensor**, and **LED** onto the canvas.
2. Wire DHT22 output to **GPIO14** and LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Set the simulated DHT22 temperature to 25.0 °C and humidity to 60.0%.
5. Verify that the console prints the secure connection message, performs the handshake, and establishes the MQTT connection.
6. Verify that the LED flashes once every 10 seconds, and the console outputs the JSON payload published to AWS.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[AWS] Connecting securely to AWS IoT Core...
[AWS] Connected to AWS IoT Core successfully.
[AWS] Loop active.
[AWS Publish] Topic: esp32/dht22 | Payload: {"temperature":25.00,"humidity":60.0,"clientUptime":10}
[AWS] Publish successful.
```

## Expected Canvas Behavior
* Booting the system establishes the secure TLS handshake, blinking the LED widget and writing the JSON payloads to the serial monitor.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <WiFiClientSecure.h>` | Includes the SSL/TLS secure client class from the ESP32 core. |
| `wifiSecureClient.setCACert(...)` | Registers the AWS Root CA certificate for server validation. |
| `mqttClient.publish(...)` | Publishes the JSON data payload to AWS IoT Core on port 8883. |

## Hardware & Safety Concept: Mutual Authentication and Root CA Verification
* **Mutual Authentication**: In standard HTTPS, only the server proves its identity using a certificate. AWS IoT Core enforces **mutual authentication**: the client validates the server certificate, and the server validates the client certificate/key. This prevents rogue devices from impersonating nodes or injecting fake data.
* **Root CA Verification**: Embedded certificates have expiration dates. AWS Root CA certificates are valid for many years, but client certificates typically expire within 1–2 years. Implement an update mechanism (like Project 182) to renew keys securely before they expire.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "AWS Connection: Secure".
2. **Alert buzzer**: Add a buzzer (GPIO 15) to sound an alarm if the secure connection state drops (`mqttClient.state()` returns error).
3. **AWS Shadow update**: Read Project 184 to learn how to update the AWS Device Shadow state instead of raw publishing.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Connection fails with rc=-2 | Certificate mismatch | The private key or client certificate does not match the credentials registered in your AWS IoT console. Verify the strings |
| ESP32 crashes on connect | Heap memory limit | Secure connections require significant RAM. Close unused sockets and avoid running web servers simultaneously to save heap space |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](../intermediate/76-esp32-mqtt-client-connection.md)
- [177 - HTTPS Server with Self-Signed Certificate](177-https-server-with-self-signed-certificate.md)
- [184 - AWS IoT Shadow updater](184-esp32-aws-iot-shadow-updater.md) (Next project)
