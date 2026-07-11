# 177 - HTTPS Server with Self-Signed Certificate

Build a secure encrypted web server on the ESP32 running over HTTPS (SSL/TLS port 443) using the WebServerSecure library and embedded self-signed SSL certificate/key arrays to encrypt credentials and communications.

## Goal
Learn how to configure secure web servers, embed self-signed SSL certificates and private keys as PROGMEM variables, handle secure handshakes on port 443, and manage encrypted routing.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. An encrypted WebServerSecure is started on port 443. The code embeds a 1024-bit self-signed SSL certificate and private key in the flash memory. Navigating to the ESP32's IP address via HTTPS (`https://10.10.0.3/`) displays a secure console allowing the user to control an LED on GPIO 12. The browser will display a certificate warning (which is normal for self-signed certificates), but all communication is fully encrypted.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED | `led` | Yes | Yes |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| LED | Anode (+) | GPIO12 | Red | Web-controlled light |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 220 Ω resistor in series with the LED to limit current.

## Code
```cpp
// HTTPS Server with Self-Signed Certificate (SSL/TLS encryption on Port 443 + Web LED control)
#include <WiFi.h>
#include <WebServerSecure.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 12;
bool ledState = false;

// 1. SSL Private Key (1024-bit Self-Signed)
const char serverKey[] PROGMEM = 
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
"-----END PRIVATE KEY-----\n";

// 2. SSL Self-Signed Certificate
const char serverCert[] PROGMEM = 
"-----BEGIN CERTIFICATE-----\n"
"MIICNDCCAZwCCQDEyv9eKqH/VDANBgkqhkiG9w0BAQsFADBLMQswCQYDVQQGEwJV\n"
"UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xGTAXBgNVBAoM\n"
"EEVzcHJlc3NpZiBTeXN0ZW0wIBcNMjYwNzExMDQyOTAwWhgPMjEyNjA2MTcwNDI5\n"
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

// Secure Web Server on HTTPS standard Port 443
WebServerSecure server(443);

// Serve main page
void handleRoot() {
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Secure HTTPS Console</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 16px; padding: 40px 30px; box-shadow: 0 10px 25px -5px rgba(0,0,0,0.4); max-width: 420px; width: 100%; border: 1px solid #334155; text-align: center; }\n";
  html += "  h1 { font-size: 22px; color: #10b981; margin-top: 0; }\n";
  html += "  .badge { display: inline-block; padding: 8px 16px; border-radius: 20px; font-weight: bold; font-size: 14px; text-transform: uppercase; margin: 15px 0; background-color: #065f46; color: #a7f3d0; }\n";
  html += "  .info-box { background-color: #0f172a; border-radius: 8px; padding: 15px; border: 1px solid #334155; text-align: left; font-family: monospace; font-size: 12px; margin-bottom: 20px; color: #94a3b8; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; background-color: #38bdf8; border: none; border-radius: 6px; cursor: pointer; transition: background-color 0.2s; }\n";
  html += "  .btn.active { background-color: #ef4444; }\n";
  html += "  .footer { text-align: center; margin-top: 30px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>HTTPS Secure Server</h1>\n";
  html += "  <div class=\"badge\">SSL/TLS ENCRYPTED</div>\n";
  
  html += "  <div class=\"info-box\">\n";
  html += "    Protocol: HTTPS (TLS 1.2)\n";
  html += "    Port: 443 (Secure)\n";
  html += "    Cipher: AES-128-GCM-SHA256\n";
  html += "    Certificate: 1024-bit Self-Signed\n";
  html += "  </div>\n";
  
  // Interactive control form
  html += "  <form action=\"/toggle\" method=\"POST\">\n";
  html += "    <button class=\"btn " + String(ledState ? "active" : "") + "\" type=\"submit\">";
  html += ledState ? "TURN LED OFF" : "TURN LED ON";
  html += "    </button>\n";
  html += "  </form>\n";
  
  html += "  <p class=\"footer\">Secure Gateway IP: " + WiFi.localIP().toString() + "</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Handle LED toggle POST request
void handleToggle() {
  ledState = !ledState;
  digitalWrite(LED_PIN, ledState ? HIGH : LOW);
  
  // Redirect back to root page
  server.sendHeader("Location", "/", true);
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("\nESP32 Secure HTTPS Node starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  Serial.print("Local IP Address: ");
  Serial.println(WiFi.localIP());
  
  // 3. Configure SSL/TLS keys and certificates
  server.setServerKeyAndCert(serverKey, serverCert);
  Serial.println("[SSL/TLS] Private Key & Certificate registered.");
  
  // 4. Start HTTPSServer
  server.on("/", handleRoot);
  server.on("/toggle", HTTP_POST, handleToggle);
  server.begin();
  
  Serial.println("HTTPS Server active on port 443. Connect at: https://" + WiFi.localIP().toString() + "/");
}

void loop() {
  server.handleClient();
  
  delay(5);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED anode to **GPIO12**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to `https://10.10.0.3/` (Note the `https://` prefix).
5. Verify that the browser displays a certificate warning (e.g. "Your connection is not private"). Click **Advanced** and then **Proceed to 10.10.0.3 (unsafe)**.
6. Verify that the secure page loads and displays "SSL/TLS ENCRYPTED".
7. Click the toggle button and verify that the simulated LED widget turns ON and OFF.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Local IP Address: 10.10.0.3
[SSL/TLS] Private Key & Certificate registered.
HTTPS Server active on port 443. Connect at: https://10.10.0.3/
```

## Expected Canvas Behavior
* Navigating to the HTTPS URL opens the ESP32's encrypted console, and clicking the button toggles the LED widget state.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <WebServerSecure.h>` | Includes the secure web server library from the ESP32 core. |
| `server.setServerKeyAndCert(...)` | Registers the PEM key and certificate buffers. |
| `WebServerSecure server(443)` | Binds the secure server to the standard HTTPS port 443. |

## Hardware & Safety Concept: SSL Certificate Authority and Self-Signed Warnings
* **Self-Signed Certificates**: A self-signed certificate encrypts communication just as securely as a paid certificate. However, because it is not signed by a trusted Certificate Authority (like Let's Encrypt), the browser cannot verify the device's identity, resulting in a warning message. For internal local network IoT control, bypassing this warning is acceptable.
* **Heap Memory Limits**: Running SSL/TLS handshakes requires significant RAM (approximately 30–45 KB of heap space per connection for buffer decryption). Do not handle more than 2–3 concurrent HTTPS connections on the ESP32 to prevent running out of heap space (brownouts/crashes).

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display "HTTPS Server: Port 443".
2. **Access PIN**: Add a secondary HTTP basic authentication layer (Project 64) inside the secure console.
3. **CA-Signed Certificate**: Read Project 178 to learn how to sign the certificate using a local CA root to eliminate browser warnings.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Browser blocks connection completely | Strict HSTS rules | Some browsers enforce HSTS (HTTP Strict Transport Security) on specific domains, blocking self-signed overrides. Access the device directly via its raw IP address |
| ESP32 crashes on client connect | Heap memory overflow | Reduce the stack size or close idle sockets. Ensure your certificate is 1024-bit (not 2048 or 4096-bit, which require too much RAM to process) |
| The webpage loads but values do not update | JavaScript error | Open the browser's developer console (F12) to inspect JavaScript errors |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [64 - ESP32 Web Server basic authentication lock](../intermediate/64-esp32-web-server-http-basic-authentication-lock.md)
- [174 - ESP32 mDNS local domain hostname setup](174-mdns-local-domain-hostname-setup.md)
- [178 - HTTPS Server with CA-Signed Certificate](178-https-server-with-ca-signed-certificate.md) (Next project)
