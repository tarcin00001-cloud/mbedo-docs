# 73 - ESP32 DHT22 Temperature LCD HUD

Build an environmental monitoring station that reads temperature and humidity from a DHT22 sensor and displays both values on a 16x2 I2C LCD.

## Goal
Learn how to display real-time sensor data on an LCD screen using I2C communications, converting float variables to formatted strings for a user interface.

## What You Will Build
A DHT22 sensor connected to GPIO 4. The ESP32 reads the current temperature and humidity every 2 seconds, format-prints the measurements, and displays them on a 16x2 I2C LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp & Humidity input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| I2C LCD | VCC | 5V (Vin) | Red | LCD power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Share the ESP32 ground pin to connect both the DHT22 GND and LCD GND. The LCD must be connected to 5V (Vin) for the backlight to function properly, while the DHT22 runs reliably on 3.3V.

## Code
```cpp
// DHT22 Temperature LCD HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

#define DHTPIN 4
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);
  
  // Initialise LCD
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  lcd.setCursor(0, 1);
  lcd.print("Booting up...");
  
  dht.begin();
  delay(1500);
  lcd.clear();
}

void loop() {
  // Query sensor (wait 2 seconds between updates)
  delay(2000);
  
  float hum = dht.readHumidity();
  float temp = dht.readTemperature();
  
  if (isnan(hum) || isnan(temp)) {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!   ");
    lcd.setCursor(0, 1);
    lcd.print("Check Wiring    ");
    return;
  }
  
  // Format Row 1: Temp
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temp, 1);
  lcd.print((char)223); // Degree symbol in LCD character set
  lcd.print("C   ");
  
  // Format Row 2: Humidity
  lcd.setCursor(0, 1);
  lcd.print("Humidity: ");
  lcd.print(hum, 1);
  lcd.print("%  ");
  
  Serial.print("Temp: "); Serial.print(temp, 1);
  Serial.print(" | Hum: "); Serial.println(hum, 1);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, and **16x2 I2C LCD** onto the canvas.
2. Connect DHT22 DATA to **GPIO4**, and LCD SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the DHT22 temperature and humidity sliders and watch the LCD values change.

## Expected Output
Serial Monitor:
```
Temp: 24.5 | Hum: 45.2
Temp: 26.8 | Hum: 50.0
```

LCD Display:
```
Temp: 24.5°C
Humidity: 45.2%
```

## Expected Canvas Behavior
* The LCD displays the booting message, clears, and then displays the formatted temperature on the top line and humidity on the bottom line.
* The readings refresh every 2 seconds matching the DHT22's sample window.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `lcd.print((char)223)` | Sends ASCII code 223, which displays the degree symbol (°) on HD44780 controller LCDs. |
| `lcd.print(temp, 1)` | Prints the float value of temperature rounded to 1 decimal place. |
| `isnan(...)` | If the sensor is disconnected, displays an error screen to notify the user. |

## Hardware & Safety Concept: Liquid Crystal Character Generators
Hobby LCD modules (like the HD44780) store a built-in character set map in ROM (similar to ASCII). The degree character (°) is mapped to index `223` in standard character ROM arrays. Using `(char)223` bypasses unicode print issues in C++ and outputs the glyph directly.

## Try This! (Challenges)
1. **Heat Index Toggle**: Add a button on GPIO 15 that toggles the second row between displaying humidity and displaying the calculated heat index.
2. **Min/Max Memory HUD**: Display the minimum and maximum temperature recorded on the screen on a rotating 5-second interval.
3. **Fahrenheit Toggle Switch**: Connect a slide switch to GPIO 12. If active, display temperature in Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD display only shows black blocks | Contrast setting is incorrect | Adjust the contrast potentiometer on the back of the I2C backpack |
| LCD displays "Sensor Error!" | DATA line disconnected or floating | Check wiring, verify GPIO 4 connection, and verify pull-up resistor |
| Temperature digits overwrite and leave residual text | String clearing missing | Append trailing spaces after units (e.g. `lcd.print("C   ")`) to clear longer digits |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](71-esp32-dht22-temp-humidity-serial-logs.md)
- [74 - ESP32 DHT22 Temperature OLED Graph](74-esp32-dht22-temperature-oled-graph.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
