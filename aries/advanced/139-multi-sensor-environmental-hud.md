# 139 - Multi-Sensor Environmental HUD

Interconnect a DHT22 humidity/temperature sensor and a BMP180 pressure/altitude sensor to display complete real-time atmospheric telemetry on a 16x2 I2C LCD using the VEGA ARIES v3 board.

## Goal
Learn how to query multiple diverse sensors (DHT22 on single-bus, BMP180 on I2C) in a single system, utilize I2C bus sharing for the LCD and pressure sensor, and implement a page-swapping display manager using non-blocking timers.

## What You Will Build
A comprehensive environmental Heads-Up Display (HUD). The board continuously reads data from the DHT22 and BMP180 sensors. An I2C LCD cycles between two screens every 2 seconds:
- **Screen 1**: Displays DHT22 Temperature (in °C) and Relative Humidity (in %).
- **Screen 2**: Displays BMP180 Barometric Pressure (in hPa) and estimated Altitude (in meters).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Pressure Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC | 3V3 | Red | Sensor power (3.3V) |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Orange | Digital Single-Bus Data |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| BMP180 Sensor | VCC | 3V3 | Orange | Shared 3.3V rail |
| BMP180 Sensor | GND | GND | Grey | Shared GND rail |
| BMP180 Sensor | SDA | SDA0 (GP17) | Blue | I2C Data line (Shared) |
| BMP180 Sensor | SCL | SCL0 (GP16) | Yellow | I2C Clock line (Shared) |
| I2C LCD Display | VCC | 5V | Red | Primary 5V power |
| I2C LCD Display | GND | GND | Black | Shared GND rail |
| I2C LCD Display | SDA | SDA0 (GP17) | Blue | I2C Data line (Shared) |
| I2C LCD Display | SCL | SCL0 (GP16) | Yellow | I2C Clock line (Shared) |

> **Wiring tip:** The BMP180 and I2C LCD share the same I2C0 bus on GP17 (SDA0) and GP16 (SCL0). This is possible because each device has a unique I2C address (LCD at `0x27`, BMP180 at `0x77`).

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

#define DHTPIN 12
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

unsigned long lastPageSwitch = 0;
int currentPage = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Initialize I2C and peripherals
  Wire.begin();
  dht.begin();

  if (!bmp.begin()) {
    Serial.println("Could not find BMP180 sensor!");
  } else {
    Serial.println("BMP180 initialized.");
  }

  lcd.init();
  lcd.backlight();
  lcd.print("ENV HUD Active");
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long currentTime = millis();

  // Cycle the page display every 2000 milliseconds
  if (currentTime - lastPageSwitch >= 2000) {
    currentPage = 1 - currentPage; // Toggle between 0 and 1
    lcd.clear();

    if (currentPage == 0) {
      // Screen 1: Temp & Humidity
      float temp = dht.readTemperature();
      float hum = dht.readHumidity();

      lcd.setCursor(0, 0);
      lcd.print("Temp: ");
      lcd.print(temp, 1);
      lcd.print(" C");

      lcd.setCursor(0, 1);
      lcd.print("Humidity: ");
      lcd.print(hum, 1);
      lcd.print(" %");

      Serial.print("DHT Temp: "); Serial.print(temp);
      Serial.print(" C | Hum: "); Serial.print(hum);
      Serial.println(" %");
    } else {
      // Screen 2: Barometric Pressure & Altitude
      float pressure = bmp.readPressure() / 100.0; // Convert Pa to hPa
      float altitude = bmp.readAltitude();

      lcd.setCursor(0, 0);
      lcd.print("Pres: ");
      lcd.print(pressure, 1);
      lcd.print(" hPa");

      lcd.setCursor(0, 1);
      lcd.print("Altitude: ");
      lcd.print(altitude, 1);
      lcd.print(" m");

      Serial.print("BMP Pressure: "); Serial.print(pressure);
      Serial.print(" hPa | Alt: "); Serial.print(altitude);
      Serial.println(" m");
    }

    lastPageSwitch = currentTime;
  }
  
  delay(10); // Maintain short loop execution
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **DHT22 Sensor**, **BMP180 Sensor**, and **I2C LCD Display** onto the canvas.
2. Wire the DHT22: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
3. Wire the BMP180: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Wire the I2C LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the program.
8. Modify DHT22 slider values and check the I2C LCD display cycle between parameters.

## Expected Output
Serial Monitor:
```
System Initialized.
BMP180 initialized.
DHT Temp: 26.30 C | Hum: 50.40 %
BMP Pressure: 1013.2 hPa | Alt: 112.5 m
DHT Temp: 26.20 C | Hum: 50.60 %
```

## Expected Canvas Behavior
* The I2C LCD backlighting turns on and displays the temperature and humidity.
* Every 2 seconds, the text flips to show the current pressure and calculated altitude.
* Adjusting the sliders changes the displayed values on the next page cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Wire.begin()` | Commences Master Mode for I2C communication on the shared lines. |
| `currentPage = 1 - currentPage` | Flip-flops the page state variable back and forth between 0 and 1. |
| `bmp.readPressure() / 100.0` | Queries the BMP180 sensor and converts standard Pascals into hectopascals. |
| `lcd.clear()` | Removes all characters from the LCD buffer before printing a new screen page. |
| `lcd.setCursor(0, 1)` | Directs subsequent prints to start at the beginning of the second LCD line. |

## Hardware & Safety Concept
* **Shared Bus Address Resolution**: Multiple devices can live on the same I2C bus because each data frame begins with a target address byte. The LCD will ignore data packets meant for the BMP180 (`0x77`) and vice-versa, allowing clean multi-device communication over just two wire lines.
* **Pull-up Resistors**: I2C lines are open-drain and require pull-up resistors (typically 4.7kΩ) to VCC. Most breakout boards (like the BMP180 or PCF8574 LCD expander) include these on-board.

## Try This! (Challenges)
1. **Third Page**: Add a third page showing the system uptime in seconds (e.g. `Uptime: 12 seconds`).
2. **Speed Page Cycle**: Modify the timing so that the pages swap every 1 second instead of 2.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD does not cycle | Code execution blocked | Verify there are no while/for loops or excessively long delay blocks in your program. |
| BMP180 init fails | Missing I2C connection | Make sure SDA is wired to GP17 and SCL is wired to GP16. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - SD Card Current/Energy Logger](138-sd-card-current-energy-logger.md) (Previous project)
- [140 - Multi-Sensor Air Quality HUD](140-multi-sensor-air-quality-hud.md) (Next project)
