# 64 - Web Server HTTP basic authentication lock

Configure a secured web server on the ESP32 that enforces HTTP Basic Authentication challenge-response checks, restricts access to the root page and toggle routes, and controls an LED on GPIO 13.

## Goal
Learn how to implement HTTP Basic Authentication, generate browser credentials challenges (HTTP code 401 Unauthorized), validate usernames and passwords, and secure IoT endpoints.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. An LED is on GPIO 13. Navigating to the ESP32's IP address prompts the browser to show a standard login credentials challenge window. Entering the correct credentials (`admin` / `password123`) loads the LED control panel. Entering incorrect credentials blocks access.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Red LED | Anode (+) | GPIO13 via 330 Ω | Red | Controlled output LED |
| Shared ground | GND | GND | Black | Shared reference |

> **Wiring tip:** Connect the Red LED anode to GPIO 13 via a 330 Ω resistor.

## Code
```cpp
// Web Server HTTP basic authentication lock (Secured Control Panel)
#include <WiFi.h>
#include <WebServer.h>

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int LED_PIN = 13;

// Configure access credentials
const char* authUsername = "admin";
const char* authPassword = "password123";

WebServer server(80);

// Root URL Route Handler ("/")
void handleRoot() {
  // 1. Challenge client credentials using HTTP Basic Auth
  // Returns false if credentials are missing or incorrect
  if (!server.authenticate(authUsername, authPassword)) {
    Serial.println("[Auth Alert] Access denied: Credentials missing or incorrect.");
    
    // Trigger standard browser login dialog (HTTP 401 challenge)
    return server.requestAuthentication();
  }
  
  Serial.println("[Auth Success] Client authenticated. Loading dashboard...");
  bool ledState = (digitalRead(LED_PIN) == HIGH);
  
  // Compile control dashboard
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>Secured LED Panel</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.5); max-width: 350px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 25px; }\n";
  html += "  .status { font-size: 18px; margin-bottom: 30px; color: #94a3b8; }\n";
  html += "  .status-val { font-weight: bold; color: " + String(ledState ? "#10b981" : "#ef4444") + "; }\n";
  html += "  .btn { display: inline-block; width: 100%; padding: 14px; font-size: 16px; font-weight: bold; color: white; border: none; border-radius: 6px; cursor: pointer; text-decoration: none; transition: background-color 0.2s; }\n";
  html += "  .btn-on { background-color: #059669; }\n";
  html += "  .btn-off { background-color: #dc2626; }\n";
  html += "  .logout-tip { margin-top: 25px; font-size: 11px; color: #64748b; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Secured LED Panel</h1>\n";
  html += "  <p class=\"status\">LED State: <span class=\"status-val\">" + String(ledState ? "ACTIVE" : "INACTIVE") + "</span></p>\n";
  
  html += "  <form action=\"/toggle\" method=\"POST\">\n";
  if (ledState) {
    html += "    <button type=\"submit\" class=\"btn btn-off\">Deactivate LED</button>\n";
  } else {
    html += "    <button type=\"submit\" class=\"btn btn-on\">Activate LED</button>\n";
  }
  html += "  </form>\n";
  html += "  <p class=\"logout-tip\">To logout, close your browser session.</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// POST Toggle Handler ("/toggle")
void handleToggle() {
  // 2. Validate authentication on write actions to prevent unauthorized API requests
  if (!server.authenticate(authUsername, authPassword)) {
    return server.requestAuthentication();
  }
  
  if (server.method() == HTTP_POST) {
    bool nextState = !digitalRead(LED_PIN);
    digitalWrite(LED_PIN, nextState ? HIGH : LOW);
    Serial.printf("[Server] Secured toggle action processed. LED: %s\n", nextState ? "ON" : "OFF");
  }
  
  server.sendHeader("Location", "/");
  server.send(303);
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start OFF
  
  Serial.println("\nESP32 Secured Web Server Starting...");
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected successfully.");
  
  // Register routes
  server.on("/", handleRoot);
  server.on("/toggle", HTTP_POST, handleToggle);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
}

void loop() {
  server.handleClient();
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **LED** onto the canvas.
2. Wire LED to **GPIO13**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address.
5. In the login window, enter `admin` and `password123`. Click the toggle button to test access.

## Expected Output
Serial Monitor:
```
ESP32 Secured Web Server Starting...
....
WiFi Connected successfully.
Web Server active. Connect at: http://10.10.0.3

[Auth Alert] Access denied: Credentials missing or incorrect.
[Auth Success] Client authenticated. Loading dashboard...
[Server] Secured toggle action processed. LED: ON
```

## Expected Canvas Behavior
* Toggling the button on the secured page switches the Red LED widget.
* Entering incorrect credentials blocks access and leaves the LED unchanged.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `server.authenticate(...)` | Compares the incoming authorization headers against target credentials. |
| `server.requestAuthentication()` | Sends a `401 Unauthorized` HTTP response containing WWW-Authenticate headers to prompt a login box. |
| `admin` / `password123` | Hardcoded credentials used to validate the client headers. |

## Hardware & Safety Concept: HTTP Basic Authentication Strengths and Limits
* **Strengths**: HTTP Basic Authentication is easy to implement and supported natively by all web browsers without requiring custom cookies or login forms.
* **Limits**: Credentials are encoded in Base64 (not encrypted) and sent in the header of every request. On unencrypted networks (HTTP), an interceptor can decode the credentials easily. For production security, combine authentication with HTTPS (SSL/TLS) (Project 33).

## Try This! (Challenges)
1. **OLED Status Display**: Add an OLED screen (Project 60) and display a padlock icon showing if the lock is active.
2. **Access Log**: Write code to print the client IP address (`server.client().remoteIP()`) to the Serial Monitor on every successful login.
3. **Buzzer alarm on failure**: Sound an alert beep (Project 15) if an authentication request fails.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Login prompt does not show | Route configuration wrong | Ensure `server.authenticate()` is called first and returns `server.requestAuthentication()` on failure |
| Valid credentials rejected | Typo in configuration | Verify that the credentials entered in the browser match `authUsername` and `authPassword` |
| Login prompt shows repeatedly | Session cookies disabled | Enable session cookies in your browser settings |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [33 - ESP32 HTTPS Secure GET Request](../beginner/33-esp32-https-secure-get-request.md)
- [44 - ESP32 Web-based LED toggle](../beginner/44-esp32-web-based-led-toggle.md)
- [65 - ESP32 Web Server Custom 404 page handler](65-esp32-web-server-custom-404-page-handler.md) (Next project)
