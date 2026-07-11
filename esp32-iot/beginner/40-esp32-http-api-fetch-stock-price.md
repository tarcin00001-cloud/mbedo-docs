# 40 - ESP32 HTTP API Fetch Stock Price

Build a financial asset price ticker on the ESP32 that connects to a public financial API, downloads live asset pricing data in JSON format, parses the value, and displays the price on a 16x2 I2C LCD.

## Goal
Learn how to query financial APIs, parse nested JSON values, format currency values, and display them on LCD screens.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). The ESP32 queries the public CoinCap API `api.coincap.io` (which requires no API key registration) for the live price of Bitcoin (BTC) in USD, parses the price string from the JSON payload, formats it as currency, and displays it on the LCD.

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
// HTTP API Fetch Asset Price (Public Financial API Ticker)
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

// Public API endpoint for asset price (Bitcoin) - Requires no API key
const char* apiURL = "https://api.coincap.io/v2/assets/bitcoin";

void fetchAssetPrice() {
  HTTPClient http;
  
  Serial.print("[HTTP] Fetching asset price from: ");
  Serial.println(apiURL);
  
  // 1. Initialize HTTP Client connection
  if (!http.begin(apiURL)) {
    Serial.println("[HTTP] Connection failed!");
    return;
  }
  
  // 2. Transmit GET request
  int httpCode = http.GET();
  
  if (httpCode == HTTP_CODE_OK) {
    // 3. Read response payload string
    String payload = http.getString();
    
    // 4. Allocate JSON Document buffer (1024 bytes is sufficient)
    DynamicJsonDocument doc(1024);
    
    // 5. Deserialize the JSON document
    DeserializationError error = deserializeJson(doc, payload);
    
    if (!error) {
      // 6. Extract values from nested JSON
      const char* name = doc["data"]["name"];
      const char* symbol = doc["data"]["symbol"];
      const char* priceUsdStr = doc["data"]["priceUsd"];
      
      // Convert price string to float
      float priceUsd = atof(priceUsdStr);
      
      Serial.println("[JSON] Parsing successful!");
      Serial.printf("Asset: %s (%s)\n", name, symbol);
      Serial.printf("Price: $%.2f USD\n", priceUsd);
      
      // 7. Display asset price on LCD
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print(name);
      lcd.print(" ("); lcd.print(symbol); lcd.print(")");
      
      lcd.setCursor(0, 1);
      lcd.print("Price: $");
      lcd.print(priceUsd, 2);
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
  lcd.print("Asset Ticker");
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
  fetchAssetPrice();
}

void loop() {
  static unsigned long lastUpdate = 0;
  unsigned long now = millis();
  
  // Refresh price data every 30 seconds
  if (now - lastUpdate >= 30000) {
    if (WiFi.status() == WL_CONNECTED) {
      fetchAssetPrice();
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
4. Watch the LCD screen initialize, connect to WiFi, and display the live asset price.

## Expected Output
Serial Monitor:
```
[HTTP] Fetching asset price from: https://api.coincap.io/v2/assets/bitcoin
[JSON] Parsing successful!
Asset: Bitcoin (BTC)
Price: $58240.50 USD
[HTTP] Connection closed.
```

LCD Display:
```
Bitcoin (BTC)
Price: $58240.50
```

## Expected Canvas Behavior
* The LCD display initializes and updates the local asset price every 30 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `doc["data"]["priceUsd"]` | Extracts the price string from the JSON payload. |
| `atof(...)` | Converts the price string into a float value for formatting. |
| `lcd.print(priceUsd, 2)` | Prints the price value with 2 decimal places. |

## Hardware & Safety Concept: RESTful APIs and Request Limits
Financial APIs update asset prices in real time. If the ESP32 queries the API in every execution loop, the server will block the client's IP address (rate limiting). To prevent this, implement a **refresh interval** using `millis()` to limit queries to once every 30 seconds or 1 minute.

## Try This! (Challenges)
1. **OLED Price Ticker**: Add an OLED screen (Project 60) and display a graph of price changes over time.
2. **Price Alert Alarm**: Sound a buzzer (GPIO 4) if the price exceeds a target threshold.
3. **Multi-asset rotation**: Program the ESP32 to cycle between displaying the prices of Bitcoin, Ethereum, and Litecoin.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| API returns error code | URL format error | Verify that the target API endpoint is typed correctly |
| Price displays as 0.00 | Data conversion error | Ensure `atof` is used to convert the price string into a float |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [38 - ESP32 HTTP JSON API Parser](38-esp32-http-json-api-parser.md)
- [41 - ESP32 HTTP API Fetch crypto price](41-esp32-http-api-fetch-crypto-price.md) (Next project)
