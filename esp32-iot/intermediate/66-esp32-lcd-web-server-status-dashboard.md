# 66 - LCD Web Server Status Dashboard

Build a local web server on the ESP32 that hosts a control page, tracks incoming HTTP client request counts, and displays the active IP address and request statistics on a 16x2 I2C LCD.

## Goal
Learn how to track query statistics, update 16x2 LCDs in real time, monitor WiFi connection states, and display device diagnostics on hardware interfaces.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network and hosts a web server on port 80. A 16x2 I2C LCD is on I2C (GPIO 21/22). Navigating to the ESP32's IP address loads a webpage. The LCD displays the server's local IP address and increments a request counter in real time every time the page is loaded.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the LCD from the 5V Vin rail.

## Code
```cpp
// LCD Web Server Status dashboard (Dynamic query counter + LCD readout)
#include <WiFi.h>
#include <WebServer.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

WebServer server(80);

// Request tracking variables
unsigned long requestCounter = 0;
unsigned long lastLCDUpdate = 0;

// Root URL Route Handler ("/")
void handleRoot() {
  // 1. Increment request counter
  requestCounter++;
  Serial.printf("[Server] Request #%lu received from client.\n", requestCounter);
  
  // Compile HTML response page
  String html = "<!DOCTYPE html>\n<html>\n<head>\n";
  html += "<meta name=\"viewport\" content=\"width=device-width, initial-scale=1.0\">\n";
  html += "<title>ESP32 Local Server</title>\n";
  html += "<style>\n";
  html += "  body { font-family: 'Segoe UI', Arial, sans-serif; background-color: #0f172a; color: #f8fafc; margin: 0; padding: 20px; display: flex; justify-content: center; align-items: center; min-height: 100vh; }\n";
  html += "  .card { background-color: #1e293b; border-radius: 12px; padding: 30px; box-shadow: 0 10px 15px -3px rgba(0, 0, 0, 0.4); max-width: 350px; width: 100%; text-align: center; border: 1px solid #334155; }\n";
  html += "  h1 { font-size: 22px; color: #38bdf8; margin-bottom: 15px; }\n";
  html += "  .count { font-size: 40px; font-weight: 800; color: #10b981; margin: 20px 0; }\n";
  html += "</style>\n</head>\n<body>\n";
  
  html += "<div class=\"card\">\n";
  html += "  <h1>Request Tracked!</h1>\n";
  html += "  <p>The LCD screen on the device has updated immediately.</p>\n";
  html += "  <div class=\"count\">#" + String(requestCounter) + "</div>\n";
  html += "  <p style=\"font-size: 13px; color: #64748b;\">Refresh the page to increment the counter.</p>\n";
  html += "</div>\n";
  
  html += "</body>\n</html>\n";
  
  server.send(200, "text/html", html);
}

// Update the 16x2 LCD status screen
void updateLCDDisplay() {
  lcd.clear();
  
  // 2. Check WiFi connection status
  if (WiFi.status() == WL_CONNECTED) {
    // Row 0: Print Server Local IP Address
    lcd.setCursor(0, 0);
    lcd.print("IP: ");
    lcd.print(WiFi.localIP());
    
    // Row 1: Print request stats
    lcd.setCursor(0, 1);
    lcd.print("Req: ");
    lcd.print(requestCounter);
  } else {
    // WiFi offline status indicator
    lcd.setCursor(0, 0);
    lcd.print("WiFi: OFFLINE");
    lcd.setCursor(0, 1);
    lcd.print("Reconnecting...");
  }
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  // Initialize LCD screen
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Web Server Setup");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  Serial.println("\nWiFi Connected.");
  
  // Register routes
  server.on("/", handleRoot);
  
  server.begin();
  Serial.print("Web Server active. Connect at: http://");
  Serial.println(WiFi.localIP());
  
  // Print initial stats to LCD
  updateLCDDisplay();
}

void loop() {
  server.handleClient();
  
  // 3. Update the LCD screen periodically (every 1 second) to prevent lag
  unsigned long now = millis();
  if (now - lastLCDUpdate >= 1000) {
    updateLCDDisplay();
    lastLCDUpdate = now;
  }
  
  delay(2);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Open your web browser and navigate to the printed IP address. Refresh the page several times and verify that the request counter on the LCD increments.

## Expected Output
Serial Monitor:
```
WiFi Connected.
Web Server active. Connect at: http://10.10.0.3
[Server] Request #1 received from client.
[Server] Request #2 received from client.
```

LCD Display:
```
IP: 10.10.0.3
Req: 2
```

## Expected Canvas Behavior
* Refreshing the webpage increments the `Req` counter displayed on the LCD widget.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `requestCounter++` | Increments the request counter variable on every root page request. |
| `WiFi.localIP()` | Queries the WiFi stack for the assigned DHCP IP address. |
| `now - lastLCDUpdate >= 1000` | Limits the LCD write frequency to 1 Hz, keeping client handling fast. |

## Hardware & Safety Concept: Separating Server Processing from LCD Writing
Writing data to I2C displays (like LCDs or OLEDs) involves transmitting bytes over serial buses. If a web server writes to the LCD inside the client request route handler (e.g. `handleRoot()`), it blocks the request loop, increasing connection latency for the client. By writing to the LCD asynchronously inside the main loop at a limited interval (e.g. 1 Hz), the server remains responsive.

## Try This! (Challenges)
1. **OLED Status Display**: Upgrade the screen to an SSD1306 OLED (Project 60) and display a graph of request history over time.
2. **Error Route Tracker**: Add route `/error` and show the count of error requests on the LCD.
3. **Buzzer alert on join**: Sound a quick beep on a buzzer (GPIO 15) when a request is tracked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| Request count does not update | Page cached by browser | Use Shift+F5 to force the browser to reload the page without using cached assets |
| Web page responds slowly | Blocking delays | Avoid using blocking `delay()` calls inside the main loop |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [37 - ESP32 LCD Network Clock](../beginner/37-esp32-lcd-network-clock.md)
- [51 - ESP32 Single Page HTTP Web Server](51-esp32-single-page-http-web-server.md)
- [67 - ESP32 Web Page DHT22 climate display](67-esp32-web-page-dht22-climate-display.md) (Next project)
