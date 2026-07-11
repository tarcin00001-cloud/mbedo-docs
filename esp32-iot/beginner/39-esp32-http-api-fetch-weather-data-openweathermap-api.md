# 39 - ESP32 HTTP API Fetch Weather Data (OpenWeatherMap API)

Build an online weather dashboard on the ESP32 that connects to a public weather API, downloads live temperature and wind speed parameters in JSON format, parses the data, and displays it on an I2C LCD.

## Goal
Learn how to query real-world public web APIs, parse nested JSON responses, handle network timeouts, and display data on I2C LCDs.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). The ESP32 queries the open meteorological server `api.open-meteo.com` (which requires no API key registration) for the current weather coordinates of Bangalore, parses the temperature and wind speed from the JSON payload, and displays them on the LCD.

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
// HTTP API Fetch Weather Data (Public Web API Query)
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Query weather coordinates for Bangalore, India
// current_weather=true returns current temperature and wind speed parameters
const char* weatherURL = "http://api.open-meteo.com/v1/forecast?latitude=12.9716&longitude=77.5946&current_weather=true";

void fetchWeather() {
  HTTPClient http;
  
  Serial.print("[HTTP] Fetching weather from: ");
  Serial.println(weatherURL);
  
  // 1. Initialize HTTP Client connection
  if (!http.begin(weatherURL)) {
    Serial.println("[HTTP] Connection failed!");
    return;
  }
  
  // 2. Transmit GET request
  int httpCode = http.GET();
  
  if (httpCode == HTTP_CODE_OK) {
    // 3. Read response payload string
    String payload = http.getString();
    
    // 4. Allocate JSON Document buffer (512 bytes is sufficient)
    StaticJsonDocument<512> doc;
    
    // 5. Deserialize the JSON document
    DeserializationError error = deserializeJson(doc, payload);
    
    if (!error) {
      // 6. Extract weather values from nested JSON
      float temperature = doc["current_weather"]["temperature"];
      float windspeed = doc["current_weather"]["windspeed"];
      
      Serial.println("[JSON] Parsing successful!");
      Serial.print("Temperature: "); Serial.print(temperature); Serial.println(" C");
      Serial.print("Wind Speed:  "); Serial.print(windspeed); Serial.println(" km/h");
      
      // 7. Display weather data on LCD
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("BLR Temp: "); lcd.print(temperature, 1); lcd.print(" C");
      lcd.setCursor(0, 1);
      lcd.print("Wind: "); lcd.print(windspeed, 1); lcd.print(" km/h");
    } else {
      Serial.print("[JSON] Parsing failed! Error: ");
      Serial.println(error.c_str());
    }
  } else {
    Serial.print("[HTTP] GET failed! Error code: ");
    Serial.println(httpCode);
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("API Read Error!");
  }
  
  // 8. Release resources
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  lcd.setCursor(0, 1);
  lcd.print("Connecting WiFi");
  
  WiFi.mode(WIFI_STA);
  WiFi.begin(ssid, password);
  
  while (WiFi.status() != WL_CONNECTED) {
    delay(500);
    Serial.print(".");
  }
  
  lcd.clear();
  lcd.print("WiFi Connected.");
  delay(1000);
  
  // Initial fetch
  fetchWeather();
}

void loop() {
  static unsigned long lastUpdate = 0;
  unsigned long now = millis();
  
  // Refresh weather data every 30 seconds
  if (now - lastUpdate >= 30000) {
    if (WiFi.status() == WL_CONNECTED) {
      fetchWeather();
    } else {
      Serial.println("WiFi offline!");
    }
    lastUpdate = now;
  }
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **16x2 I2C LCD** onto the canvas.
2. Wire LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Watch the LCD screen initialize, connect to WiFi, and display the live temperature and wind speed.

## Expected Output
Serial Monitor:
```
[HTTP] Fetching weather from: http://api.open-meteo.com/v1/forecast?latitude=12.9716&longitude=77.5946&current_weather=true
[JSON] Parsing successful!
Temperature: 28.4 C
Wind Speed:  12.5 km/h
[HTTP] Connection closed.
```

LCD Display:
```
BLR Temp: 28.4 C
Wind: 12.5 km/h
```

## Expected Canvas Behavior
* The LCD display initializes and updates the local weather data every 30 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `doc["current_weather"]["temperature"]` | Resolves nested objects to extract the temperature value from the JSON payload. |
| `lcd.print(temperature, 1)` | Prints the temperature value with 1 decimal place. |
| `http.end()` | Closes sockets and releases memory. |

## Hardware & Safety Concept: RESTful APIs and API Key Security
Many online weather services (like OpenWeatherMap) require registering for an API key to limit request counts. When writing code:
1. **API Keys**: Store API keys in separate configurations or headers rather than hardcoding them in shared source files.
2. **Request limits**: Limit the frequency of queries (e.g. once every 30 seconds or 5 minutes) to avoid getting your IP banned or rate-limited.

## Try This! (Challenges)
1. **OLED Weather HUD**: Add an OLED screen (Project 60) and display a weather icon (e.g. sun or cloud) based on the weather conditions.
2. **Temperature Alert**: Sound a buzzer (GPIO 4) if the temperature exceeds a safety limit (e.g. 35.0 °C).
3. **Multi-city rotation**: Program the ESP32 to cycle between displaying the weather for London, New York, and Tokyo.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| API returns error code | URL format error | Verify that the coordinate query parameters are typed correctly |
| The ESP32 crashes on loop | Stack size overflow | Ensure the JSON document buffer size is sufficient to hold the parsed JSON tree |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [38 - ESP32 HTTP JSON API Parser](38-esp32-http-json-api-parser.md)
- [37 - ESP32 LCD Network Clock](37-esp32-lcd-network-clock.md)
