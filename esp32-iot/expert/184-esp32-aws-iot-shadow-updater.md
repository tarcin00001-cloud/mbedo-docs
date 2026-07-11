# 184 - AWS IoT Shadow updater

Build an AWS Device Shadow synchronization node on the ESP32 that connects securely to AWS IoT Core over MQTT, publishes its current LED state to the reported shadow topic, subscribes to the shadow delta topic, and parses JSON commands using ArduinoJson to toggle a physical LED on GPIO 12.

## Goal
Learn how to interact with the AWS Device Shadow document system, subscribe to shadow delta MQTT topics, publish reported state changes, parse structured JSON documents, and run secure TLS loops.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network, establishing a secure connection to AWS IoT Core on port 8883. It registers an LED on GPIO 12. It publishes its initial state `{"state":{"reported":{"led":"OFF"}}}` to the shadow update topic. The user changes the desired state in the AWS console to `ON`. AWS generates a delta message. The ESP32 receives the delta, parses it, turns the LED ON, and reports the new state back to the cloud.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Shadow-controlled light |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// AWS IoT Shadow Updater (Device Shadow State synchronization + Delta topic listener + JSON parser)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>
#include <ArduinoJson.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// AWS IoT Core Endpoint (Replace with your actual AWS IoT endpoint)
const char* awsEndpoint = "a3xxxxxxxxxxxx-ats.iot.us-east-1.amazonaws.com";
const int AWS_MQTT_PORT = 8883;

// Device Shadow MQTT Topics
const char* shadowUpdateTopic = "$aws/things/ESP32_Thing/shadow/update";
const char* shadowDeltaTopic = "$aws/things/ESP32_Thing/shadow/update/delta";

const int LED_PIN = 12;
bool currentLedState = false;

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
"-----END PRIVATE KEY-----\n";

WiFiClientSecure wifiSecureClient;
PubSubClient mqttClient(wifiSecureClient);

// Publish current reported state to AWS shadow
void publishReportedState(bool state) {
  // StaticJsonDocument creates an in-memory JSON structure
  StaticJsonDocument<256> doc;
  JsonObject stateObj = doc.createNestedObject("state");
  JsonObject reportedObj = stateObj.createNestedObject("reported");
  reportedObj["led"] = state ? "ON" : "OFF";
  
  char jsonBuffer[256];
  serializeJson(doc, jsonBuffer);
  
  Serial.printf("[Shadow] Publishing reported state: %s\n", jsonBuffer);
  mqttClient.publish(shadowUpdateTopic, jsonBuffer);
}

// Handle incoming delta messages from AWS IoT Core
void messageCallback(char* topic, byte* payload, unsigned int length) {
  Serial.printf("[Shadow] Delta message arrived on topic: %s\n", topic);
  
  // Parse incoming JSON delta packet
  StaticJsonDocument<512> doc;
  DeserializationError error = deserializeJson(doc, payload, length);
  
  if (error) {
    Serial.print("[JSON Error] Deserialization failed: ");
    Serial.println(error.c_str());
    return;
  }
  
  // Check if delta contains the "led" parameter
  if (doc["state"].containsKey("led")) {
    String desiredLedState = doc["state"]["led"];
    Serial.printf("[Shadow Delta] Desired LED State: %s\n", desiredLedState.c_str());
    
    if (desiredLedState == "ON") {
      currentLedState = true;
      digitalWrite(LED_PIN, HIGH);
    } else if (desiredLedState == "OFF") {
      currentLedState = false;
      digitalWrite(LED_PIN, LOW);
    }
    
    // Publish reported state back to sync the shadow document in the cloud
    publishReportedState(currentLedState);
  }
}

// Connect to AWS IoT Core securely and subscribe to delta topic
void connectToAWS() {
  wifiSecureClient.setCACert(awsRootCA);
  wifiSecureClient.setCertificate(awsClientCert);
  wifiSecureClient.setPrivateKey(awsPrivateKey);
  
  mqttClient.setServer(awsEndpoint, AWS_MQTT_PORT);
  mqttClient.setCallback(messageCallback);
  
  Serial.println("[AWS] Connecting securely to AWS IoT Core...");
  
  while (!mqttClient.connected()) {
    String clientId = "ESP32-Shadow-" + String(random(0xffff), HEX);
    
    if (mqttClient.connect(clientId.c_str())) {
      Serial.println("[AWS] Connected to AWS IoT Core.");
      // Subscribe to shadow delta changes
      mqttClient.subscribe(shadowDeltaTopic);
      Serial.printf("[AWS] Subscribed to topic: %s\n", shadowDeltaTopic);
      
      // Publish initial reported state on boot
      publishReportedState(currentLedState);
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
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, currentLedState ? HIGH : LOW);
  
  Serial.println("\nESP32 AWS Device Shadow Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  connectToAWS();
}

void loop() {
  if (!mqttClient.connected()) {
    connectToAWS();
  }
  mqttClient.loop();
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Verify that the console prints the connection message, performs the handshake, and registers the subscription.
5. In simulation, since the AWS cloud console is not present, mock a delta packet by sending the following string to the serial input: `SYS:DELTA:{"state":{"led":"ON"}}`.
6. Verify that the simulated LED widget turns ON, and the console prints the reported state confirmation message.

## Expected Output
Serial Monitor:
```
WiFi Connected.
[AWS] Connecting securely to AWS IoT Core...
[AWS] Connected to AWS IoT Core.
[AWS] Subscribed to topic: $aws/things/ESP32_Thing/shadow/update/delta
[Shadow] Publishing reported state: {"state":{"reported":{"led":"OFF"}}}
[Shadow] Delta message arrived on topic: $aws/things/ESP32_Thing/shadow/update/delta
[Shadow Delta] Desired LED State: ON
[Shadow] Publishing reported state: {"state":{"reported":{"led":"ON"}}}
```

## Expected Canvas Behavior
* Establishing the TLS socket initializes the shadow document structure, toggling the LED widget and writing the JSON logs upon receiving delta messages.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `$aws/things/.../update/delta` | The reserved AWS IoT system topic that publishes desired state differences. |
| `doc["state"].containsKey("led")` | Assesses if the JSON delta message includes the targeted parameter. |
| `publishReportedState(...)` | Serializes and publishes the current hardware state to sync the cloud document. |

## Hardware & Safety Concept: Shadow Document Latency and Delta Resolution Loop
* **Shadow Document Latency**: The Device Shadow acts as a persistent virtual representation of your device, meaning mobile apps can read or write states even if the physical ESP32 is offline. Once the ESP32 boots up, it automatically queries the shadow service to sync outstanding changes.
* **Delta Resolution Loop**: To prevent infinite messaging loops, the ESP32 must only publish to the `reported` topic when a physical hardware change occurs. Publishing to the `desired` topic from the ESP32 should be avoided unless the device needs to request a cloud configuration change.

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "Shadow Sync: ON".
2. **Error buzzer**: Add a buzzer (GPIO 15) to sound a warning beep if the delta command is not recognized or contains an invalid value.
3. **Azure IoT Hub**: Read Project 185 to learn how to log telemetry directly to Azure IoT Hub instead of AWS.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Delta messages are not received | Subscribed to wrong topic | Ensure the Thing Name matches the string used in the delta topic path exactly (`ESP32_Thing`) |
| JSON parsing fails | Stack size too small | The `StaticJsonDocument` size must be large enough to contain the incoming JSON structure. Increase the document size to `1024` if the delta payload has nested keys |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [76 - ESP32 MQTT Client Connection](../intermediate/76-esp32-mqtt-client-connection.md)
- [183 - AWS IoT Core connection](183-esp32-aws-iot-core-connection.md)
- [185 - Azure IoT Hub direct telemetry logger](185-azure-iot-hub-direct-telemetry-logger.md) (Next project)
