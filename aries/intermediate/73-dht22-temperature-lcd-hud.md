# 73 - DHT22 Temperature LCD HUD

Display live climate readings from a DHT22 sensor onto a 16x2 I2C Character LCD using the VEGA ARIES v3 board.

## Goal
Learn how to interface both an I2C communication peripheral and a digital sensor simultaneously, and display formatted strings on an LCD screen.

## What You Will Build
A DHT22 sensor is connected to GPIO 12, and an I2C 16x2 LCD is connected to the hardware I2C0 pins. The LCD shows the live temperature in Celsius on the top row and relative humidity on the bottom row.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | Power input |
| DHT22 Sensor | DATA (Pin 2) | GPIO 12 | Yellow | Bidirectional data line |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground reference |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** SDA0 and SCL0 correspond to the default I2C0 interface on the ARIES v3 board (pins GP17 and GP16 respectively). Verify that both VCC and GND connections are solid.

## Code
```cpp
// DHT22 Temperature LCD HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

#define DHTPIN 12
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.begin(); // Initialize default I2C0 interface
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("System Initialized");
  
  dht.begin();
  delay(1000);
  lcd.clear();
}

void loop() {
  // Query sensor every 2 seconds
  delay(2000);
  
  float humidity = dht.readHumidity();
  float temperature = dht.readTemperature();
  
  if (isnan(humidity) || isnan(temperature)) {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error    ");
    lcd.setCursor(0, 1);
    lcd.print("Check Wiring    ");
    return;
  }
  
  // Display Temperature on Row 1
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(temperature, 1);
  lcd.print(" C     "); // Trailing spaces to overwrite old text
  
  // Display Humidity on Row 2
  lcd.setCursor(0, 1);
  lcd.print("Humid: ");
  lcd.print(humidity, 1);
  lcd.print(" %     ");
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, and **I2C LCD Display** components onto the canvas.
2. Wire the DHT22: **DATA** to **GPIO 12**, **VCC** to **3V3**, and **GND** to **GND**.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Change the sliders on the DHT22 widget and observe the LCD updating.

## Expected Output
LCD Screen:
```
Temp: 24.5 C
Humid: 45.2 %
```

## Expected Canvas Behavior
* The LCD screen clears after setup and shows live climate measurements.
* Adjusting the DHT22 sliders changes the values on the screen after the next 2-second measurement window.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Wire.begin()` | Starts I2C clock and data line control. |
| `lcd.init()` | Triggers display driver configuration. |
| `lcd.print("Temp: ")` | Prints the static text label to the current cursor position. |
| `lcd.print(temperature, 1)` | Prints the temperature float formatted to 1 decimal place. |

## Hardware & Safety Concept
* **Liquid Crystal Display Architecture**: The LiquidCrystal_I2C library sends data packets over the 2-wire serial bus to a PCF8574 expander chip mounted on the LCD. This expander translates the I2C packets into 4-bit parallel data commands that the HD44780 LCD driver understands. Using I2C instead of a direct parallel connection saves 6 GPIO pins on the microcontroller, making hardware design much cleaner.

## Try This! (Challenges)
1. **Fahrenheit Toggle**: Connect a Push Button to GPIO 16. Modify the loop to display temperature in Fahrenheit when the button is pressed.
2. **Backlight Power Saver**: Turn off the backlight using `lcd.noBacklight()` if the temperature falls below a specified comfort level.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD is blank or shows white boxes | Contrast is misaligned | Turn the contrast potentiometer on the back of the LCD module. |
| LCD prints "Sensor Error" | DHT22 connection failure | Verify the DATA line of the DHT22 is connected to GPIO 12. |
| Compiler errors on LiquidCrystal | Missing libraries | Ensure that the LiquidCrystal_I2C library is installed or accessible. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [71 - DHT22 Temp & Humidity Serial Logs](71-dht22-temp-humidity-serial.md)
- [74 - DHT22 Temperature OLED Graph](74-dht22-temperature-oled-graph.md)
