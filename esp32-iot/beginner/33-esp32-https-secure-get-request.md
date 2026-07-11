# 33 - ESP32 HTTPS Secure GET Request

Configure the ESP32 to establish a secure encrypted HTTPS (SSL/TLS) client connection, send a secure GET request to a remote server, and log the response to the Serial Monitor.

## Goal
Learn how to use `WiFiClientSecure` and `HTTPClient` to negotiate TLS handshakes, manage certificate verification policies, and download secure web payloads.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. Every 10 seconds, it initiates a secure HTTPS connection to `https://httpbin.org/get`. It uses the `setInsecure()` bypass to establish TLS encryption without root certificate validation, and logs the response to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |

## Wiring
No external wiring is required. The ESP32 uses its built-in WiFi antenna.

## Code
```cpp
// HTTPS Secure GET Request (SSL/TLS Handshake)
#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <HTTPClient.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Secure HTTPS Target URL API endpoint
const char* secureURL = "https://httpbin.org/get";

// Production Security Option: Root CA certificate (in PEM format)
// In a real application, download this from the server's CA.
const char* rootCACertificate = \
"-----BEGIN CERTIFICATE-----\n" \
"MIIFazCCA1OgAwIBAgIRAIIQz7DSQONZRGPgu2OCwiAwDQYJKoZIhvcNAQELBQAw\n" \
"TzELMAkGA1UEBhMCVVMxKTAnBgNVBAoTIEludGVybmV0IFNlY3VyaXR5IERlc2Vz\n" \
"dGVyIENlbnRlci5vcmcxETAPBgNVBAMTCElTR1IgUm9vdCBYMTBkFw0yMTA5MDQw\n" \
"MDAwMDBaFw0zMDA5MDQwMDAwMDBaMEUxCzAJBgNVBAYTAlVTMSUwIwYDVQQKExxD\n" \
"bG91ZGZsYXJlLCBJbmMuMRkwFwYDVQQDExBDbG91ZGZsYXJlIE5ldyBDQTAgFw0y\n" \
"... (Root certificate goes here) ...\n" \
"-----END CERTIFICATE-----\n";

void performSecureGET() {
  // 1. Instantiate WiFiClientSecure
  WiFiClientSecure client;
  
  // 2. Set certificate validation policy
  // For testing, we use setInsecure() to bypass certificate checks.
  // This encrypts the data without validating the server's identity.
  client.setInsecure(); 
  
  // To use certificate validation instead, comment setInsecure() and uncomment:
  // client.setCACert(rootCACertificate);

  HTTPClient http;
  
  Serial.print("[HTTPS] Connecting securely to: ");
  Serial.println(secureURL);
  
  // 3. Initialize HTTP client over the secure socket
  bool success = http.begin(client, secureURL);
  
  if (!success) {
    Serial.println("[HTTPS] Secure connection initialization failed!");
    return;
  }
  
  Serial.println("[HTTPS] Initiating TLS handshake and GET request...");
  
  // 4. Send GET request
  int httpResponseCode = http.GET();
  
  // 5. Evaluate Response
  if (httpResponseCode > 0) {
    Serial.print("[HTTPS] Secure Response Status Code: ");
    Serial.println(httpResponseCode);
    
    // Retrieve response body
    String payload = http.getString();
    Serial.println("[HTTPS] --- Secure Response Body ---");
    Serial.println(payload);
    Serial.println("[HTTPS] ----------------------------");
  } else {
    Serial.print("[HTTPS] Secure GET failed! Error: ");
    Serial.println(http.errorToString(httpResponseCode).c_str());
  }
  
  // 6. Release resources
  http.end();
  Serial.println("[HTTPS] Secure connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  Serial.println("Initializing HTTPS Secure Client Test...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    performSecureGET();
  } else {
    Serial.println("WiFi offline!");
  }
  
  // Send request every 10 seconds
  delay(10000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** onto the canvas.
2. Paste the code and click **Run**.
3. Inspect the Serial Monitor. Verify that the secure response body prints.

## Expected Output
Serial Monitor:
```
Initializing HTTPS Secure Client Test...
....
WiFi Connected successfully.
[HTTPS] Connecting securely to: https://httpbin.org/get
[HTTPS] Initiating TLS handshake and GET request...
[HTTPS] Secure Response Status Code: 200
[HTTPS] --- Secure Response Body ---
{
  "args": {}, 
  "headers": {
    "Host": "httpbin.org", 
    "User-Agent": "ESP32"
  }, 
  "url": "https://httpbin.org/get"
}
[HTTPS] ----------------------------
[HTTPS] Secure connection closed.
```

## Expected Canvas Behavior
* The Serial Monitor logs the HTTPS response codes and the JSON body every 10 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `WiFiClientSecure` | Instantiates a secure TCP client socket capable of TLS negotiation. |
| `client.setInsecure()` | Configures the socket to bypass root certificate checks for testing. |
| `http.begin(client, ...)` | Binds the HTTP client helper to run over the secure socket. |
| `http.end()` | Closes secure sockets and releases memory. |

## Hardware & Safety Concept: Secure Web Communications (SSL/TLS)
Standard HTTP transmits data in plain text, making it vulnerable to interception. **HTTPS (HTTP over TLS/SSL)** encrypts the data using symmetric cryptography. The handshake involves:
1. **Key Exchange**: The client and server agree on encryption keys.
2. **Authentication (Optional)**: The client verifies the server's certificate against a trusted Certificate Authority (CA) pool to prevent spoofing.
For security, use certificate validation (`setCACert`) in production.

## Try This! (Challenges)
1. **OLED Security Dashboard**: Add an OLED screen (Project 60) and display the secure status code.
2. **Valid Certificate Check**: Download the root CA certificate for `httpbin.org` and use `setCACert()` to verify the server's identity.
3. **Buzzer confirmation tone**: Sound a quick beep on a buzzer (GPIO 4) when the TLS handshake completes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Handshake fails immediately | Out of memory | TLS negotiation requires significant RAM. Close other sockets and avoid printing in loops |
| Connection rejected with code -1 | Certificate expired | If using `setCACert`, check that the certificate has not expired or use `setInsecure()` for testing |
| Handshake times out | Weak signal | Ensure the ESP32 has a stable connection before starting the handshake |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [32 - ESP32 HTTP POST Request](32-esp32-http-post-request.md)
- [38 - ESP32 HTTP JSON API Parser](38-esp32-http-json-api-parser.md)
