# 43 - ESP32 HTTP API Currency Converter

Build an interactive currency exchange rate calculator on the ESP32 that connects to a public exchange rate API, downloads live USD exchange rates in JSON format, reads a target USD value from a potentiometer, and displays the converted EUR equivalent on an LCD.

## Goal
Learn how to parse nested currency exchange rates, implement scaling calculations on analog inputs, manage background update intervals, and build currency calculators.

## What You Will Build
An ESP32 DevKitC connects to a local WiFi network. A 16x2 I2C LCD is on I2C (GPIO 21/22). A potentiometer is on GPIO 34. The ESP32 queries the Open Exchange Rates API `open.er-api.com` (which requires no API key registration) to fetch the current USD-to-EUR exchange rate. Adjusting the potentiometer sets a USD amount (e.g. $0 to $100), and the LCD displays the converted EUR equivalent.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Potentiometer (10 kΩ) | `potentiometer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Potentiometer | Wiper | GPIO34 | Yellow | USD input adjustment |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the LCD from the 5V Vin rail.

## Code
```cpp
// HTTP API Currency Converter (USD -> EUR Live Calculator)
#include <WiFi.h>
#include <HTTPClient.h>
#include <ArduinoJson.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

LiquidCrystal_I2C lcd(0x27, 16, 2);

const char* ssid = "Wokwi-GUEST";
const char* password = "";

const int POT_PIN = 34;

// Public Exchange Rate API (returns USD exchange rates) - Requires no API key
const char* apiURL = "https://open.er-api.com/v6/latest/USD";

// Local exchange rate state variables
float usdToEurRate = 0.92; // Default fallback exchange rate
unsigned long lastRateFetch = 0;
const unsigned long FETCH_INTERVAL_MS = 60000; // Fetch rate once every 1 minute

void fetchExchangeRate() {
  HTTPClient http;
  
  Serial.print("[HTTP] Fetching exchange rates from: ");
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
    
    // 4. Allocate JSON Document buffer (1536 bytes is sufficient for currency rates map)
    DynamicJsonDocument doc(1536);
    
    // 5. Deserialize the JSON document
    DeserializationError error = deserializeJson(doc, payload);
    
    if (!error) {
      // 6. Extract target exchange rate (EUR)
      float rate = doc["rates"]["EUR"];
      
      if (rate > 0) {
        usdToEurRate = rate;
        Serial.println("[JSON] Parsing successful!");
        Serial.print("USD -> EUR Rate: "); Serial.println(usdToEurRate);
      }
    } else {
      Serial.print("[JSON] Parsing failed! Error: ");
      Serial.println(error.c_str());
    }
  } else {
    Serial.print("[HTTP] GET failed! Error code: ");
    Serial.println(httpCode);
  }
  
  // 7. Release resources
  http.end();
  Serial.println("[HTTP] Connection closed.\n");
}

void setup() {
  Serial.begin(115200);
  delay(1000);
  
  pinMode(POT_PIN, INPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Currency Conv");
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
  
  // Initial rate fetch
  fetchExchangeRate();
  lastRateFetch = millis();
}

void loop() {
  unsigned long now = millis();
  
  // 8. Periodically refresh the exchange rate in the background
  if (now - lastRateFetch >= FETCH_INTERVAL_MS) {
    if (WiFi.status() == WL_CONNECTED) {
      fetchExchangeRate();
    }
    lastRateFetch = now;
  }
  
  // 9. Read Potentiometer and map to USD amount ($0.00 to $100.00)
  int potRaw = analogRead(POT_PIN);
  float usdAmount = (float)potRaw * 100.0 / 4095.0;
  
  // Calculate EUR equivalent
  float eurAmount = usdAmount * usdToEurRate;
  
  // Display on LCD
  lcd.setCursor(0, 0);
  lcd.print("USD: $");
  lcd.print(usdAmount, 2);
  lcd.print("      "); // Clear trailing space
  
  lcd.setCursor(0, 1);
  lcd.print("EUR: ");
  lcd.write(0xE0); // Print custom character or symbol for Euro (or use 'e')
  lcd.print(eurAmount, 2);
  lcd.print("      ");
  
  // Print details to Serial Monitor less frequently
  static unsigned long lastLog = 0;
  if (now - lastLog >= 1000) {
    Serial.printf("USD: $%.2f | EUR: e%.2f (Rate: %.4f)\n", usdAmount, eurAmount, usdToEurRate);
    lastLog = now;
  }
  
  delay(100);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Potentiometer**, and **16x2 I2C LCD** onto the canvas.
2. Wire Potentiometer to **GPIO34** and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the potentiometer widget. Watch the converted currency value update on the LCD screen.

## Expected Output
Serial Monitor:
```
[HTTP] Fetching exchange rates from: https://open.er-api.com/v6/latest/USD
[JSON] Parsing successful!
USD -> EUR Rate: 0.9224
USD: $50.00 | EUR: e46.12 (Rate: 0.9224)
USD: $85.50 | EUR: e78.87 (Rate: 0.9224)
```

LCD Display:
```
USD: $50.00
EUR: e46.12
```

## Expected Canvas Behavior
* Adjusting the potentiometer updates the USD and EUR values on the LCD.
* The system fetches the latest exchange rate in the background every minute.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `doc["rates"]["EUR"]` | Extracts the EUR exchange rate from the JSON payload. |
| `usdAmount * usdToEurRate` | Multiplies the input USD amount by the active exchange rate. |
| `fetchExchangeRate()` | Connects to the API to update the exchange rate in the background. |

## Hardware & Safety Concept: Network Resource Caching
Querying public APIs consumes network bandwidth and CPU cycles. Exchange rates change slowly compared to sensor values. Running an API request in every loop execution would lag the user interface and get the device's IP banned. By caching the rate value in a variable and updating it every minute, the UI remains responsive.

## Try This! (Challenges)
1. **OLED Ticker Dashboard**: Add an OLED screen (Project 60) and display the exchange rate along with the USD and EUR amounts.
2. **Reverse Calculator Switch**: Add a button on GPIO 12 to swap the conversion direction (EUR to USD).
3. **Buzzer alert**: Sound an alert beep (Project 15) if the API request fails, informing the user of outdated rates.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Converted values display as 0.00 | Fallback rate set to 0 | Initialize the exchange rate variable with a fallback default (e.g. 0.92) |
| LCD display freezes | I2C address mismatch | Verify the I2C address in `LiquidCrystal_I2C` matches your LCD module (usually `0x27`) |
| API returns error code | URL format error | Verify that the target API endpoint is typed correctly |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [31 - ESP32 HTTP GET Request](31-esp32-http-get-request.md)
- [38 - ESP32 HTTP JSON API Parser](38-esp32-http-json-api-parser.md)
- [40 - ESP32 HTTP API Fetch stock price](40-esp32-http-api-fetch-stock-price.md)
