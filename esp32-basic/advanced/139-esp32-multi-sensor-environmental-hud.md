# 139 - ESP32 Multi-Sensor Environmental HUD

Build a multi-sensor environmental weather station that monitors temperature and humidity from a DHT22 sensor, alongside barometric pressure from a BMP180 sensor, displaying findings on a button-swappable 16x2 I2C LCD HUD.

## Goal
Learn how to read data from multiple sensor modules (one digital, one I2C), manage shared I2C buses, and build button-controlled display page menus.

## What You Will Build
A DHT22 is connected to GPIO 4. A BMP180 barometric sensor and 16x2 LCD share the I2C bus (GPIO 21/22). A push button on GPIO 15 switches between two pages: Page 1 displays DHT22 climate data, and Page 2 displays BMP180 pressure and altitude data.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Tactile Push Button | `button` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Climate digital input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| BMP180 Sensor | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (shared) |
| BMP180 Sensor | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus (shared) |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |
| Push Button | Pin 1 / Pin 2 | 3V3 / GPIO15 | Red / Green | Page switch trigger |
| 10 kΩ Resistor | Leg 1 / Leg 2 | GPIO15 / GND | White / Black | Pull-down resistor |

> **Wiring tip:** Share the I2C SDA (GPIO 21) and SCL (GPIO 22) lines for the BMP180 and I2C LCD. Wire the push button to GPIO 15 with a 10 kΩ pull-down resistor.

## Code
```cpp
// Multi-Sensor Environmental HUD
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN = 4;
const int BTN_PAGE = 15;

#define DHTTYPE DHT22
DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int currentPage = 0;
bool lastButtonState = LOW;

void setup() {
  Serial.begin(115200);
  
  pinMode(BTN_PAGE, INPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Sensors HUD Init");
  
  dht.begin();
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("BMP180 Error!");
    while(1) {}
  }
  
  delay(1500);
  lcd.clear();
}

void loop() {
  bool buttonState = (digitalRead(BTN_PAGE) == HIGH);
  
  // Detect button click (rising edge) to switch pages
  if (buttonState && !lastButtonState) {
    delay(50); // Debounce
    currentPage = (currentPage + 1) % 2;
    lcd.clear();
    Serial.print("Switched to Page: "); Serial.println(currentPage);
  }
  lastButtonState = buttonState;
  
  // Read sensor values
  float temp = dht.readTemperature();
  float hum = dht.readHumidity();
  float pressure = bmp.readPressure() / 100.0; // Convert Pa to hPa
  float altitude = bmp.readAltitude(1013.25);  // Standard sea-level reference
  
  // Update LCD based on current page
  if (currentPage == 0) {
    // Page 0: DHT22 Climate Data
    lcd.setCursor(0, 0);
    lcd.print("DHT22 Temperature");
    lcd.setCursor(0, 1);
    if (!isnan(temp) && !isnan(hum)) {
      lcd.print(temp, 1); lcd.print("C ");
      lcd.print("Hum:"); lcd.print(hum, 1); lcd.print("%  ");
    } else {
      lcd.print("Read Error...   ");
    }
  } 
  else {
    // Page 1: BMP180 Pressure Data
    lcd.setCursor(0, 0);
    lcd.print("BMP180 Pressure");
    lcd.setCursor(0, 1);
    lcd.print(pressure, 1); lcd.print("hPa ");
    lcd.print(altitude, 0); lcd.print("m    ");
  }
  
  // Log to serial monitor
  Serial.print("T: "); Serial.print(temp, 1);
  Serial.print(" C | H: "); Serial.print(hum, 1);
  Serial.print(" % | P: "); Serial.print(pressure, 1);
  Serial.print(" hPa | A: "); Serial.println(altitude, 0);
  
  delay(200); // 5Hz loop refresh
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, **BMP180**, **16x2 I2C LCD**, and **Button** onto the canvas.
2. Wire components: DHT22 to **GPIO4**, shared SDA/SCL to **GPIO21/GPIO22**, and Button to **GPIO15**.
3. Paste the code and click **Run**.
4. Observe DHT22 data on the LCD. Click the button to switch the screen and see the BMP180 pressure and altitude readings.

## Expected Output
Serial Monitor:
```
Sensors HUD Init
T: 24.5 C | H: 45.2 % | P: 1013.2 hPa | A: 0
Switched to Page: 1
T: 24.5 C | H: 45.2 % | P: 1013.2 hPa | A: 0
```

LCD Display (Page 0):
```
DHT22 Temperature
24.5C Hum:45.2%
```

LCD Display (Page 1):
```
BMP180 Pressure
1013.2hPa 0m
```

## Expected Canvas Behavior
* At boot, the LCD initializes.
* Clicking the page button widget toggles the LCD display layouts between DHT22 and BMP180 sensor data.
* Toggling sensor sliders on either the DHT22 or BMP180 widgets dynamically updates the corresponding page numbers on the LCD.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `currentPage = (currentPage + 1) % 2` | Toggles the page index between 0 and 1. |
| `bmp.readPressure() / 100.0` | Converts Pascal (Pa) readings to hectopascal (hPa). |
| `bmp.readAltitude(1013.25)` | Approximates elevation based on standard sea-level pressure. |

## Hardware & Safety Concept: shared I2C bus Address Management
I2C (Inter-Integrated Circuit) uses a 2-wire serial bus: Serial Data (SDA) and Serial Clock (SCL). Because it is an addressable protocol, multiple devices can share the same two lines. Each device must have a unique 7-bit factory-set hex address (e.g. BMP180 is usually `0x77`, LCD backpack is `0x27`). The microcontroller coordinates communication by sending the target address first, ensuring no data conflicts occur.

## Try This! (Challenges)
1. **Third page graph**: Add a third page that plots a simple scrolling history bar of temperature.
2. **Auto-Cycle Mode**: Make the HUD automatically cycle between the pages every 4 seconds if the button is not pressed.
3. **Storm warning warning**: sound a warning chime if pressure drops below 990 hPa (indicating low pressure / stormy weather).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD freezes or BMP180 initialization fails | I2C wiring issue | Check that SDA and SCL are connected to GPIO 21 and 22, and not swapped |
| Page switches continuously on one press | Missing button state latch | Ensure the code uses `buttonState && !lastButtonState` to detect single edges |
| Pressure readings are static | Sensor not responding | Check VCC voltage; ensure BMP180 is receiving 3.3V |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [73 - ESP32 DHT22 Temperature LCD HUD](../intermediate/73-esp32-dht22-temperature-lcd-hud.md)
- [79 - ESP32 BMP180 Temp & Altitude Display](../intermediate/79-esp32-bmp180-temp-altitude-display.md)
- [112 - ESP32 DS3231 Real Time Clock Display](112-esp32-ds3231-real-time-clock-display.md)
