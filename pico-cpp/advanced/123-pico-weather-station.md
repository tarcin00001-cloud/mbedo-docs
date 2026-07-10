# 123 - Pico Weather Station

Build a high-performance outdoor weather station console that combines BMP180 and DHT22 telemetry onto a graphical OLED display.

## Goal
Learn how to pool environmental data from multiple I2C and single-wire digital sensors and format a clean graphic dashboard layout on SSD1306 screens.

## What You Will Build
An advanced outdoor weather console:
- **BMP180 Sensor (GP4, GP5)**: Measures barometric pressure and altitude.
- **DHT22 Sensor (GP12)**: Measures ambient humidity and temperature.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature, humidity, pressure, and a weather trend forecast.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| All Devices | VCC | 3.3V | Shared power |
| All Devices | GND | GND | Shared ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  
  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  if (!bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("BMP180 Sensor Error!");
    display.display();
    while (1);
  }
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();
  long pressure = bmp.readPressure();

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(15, 3);
    display.print("METEO STATION HUD");

    // Display values
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Temp & Humidity
    display.setCursor(8, 20);
    display.print("T: ");
    display.print(temp, 1);
    display.print("C | H: ");
    display.print(humid, 1);
    display.print("%");

    // Row 2: Pressure
    display.setCursor(8, 34);
    display.print("Pres: ");
    display.print(pressure / 100);
    display.print(" hPa");

    // Row 3: Weather Forecast trend
    display.setCursor(8, 48);
    if (pressure < 100900) {
      display.print("Forecast: RAIN/STORM");
    } else if (pressure > 102200) {
      display.print("Forecast: SUNNY/DRY");
    } else {
      display.print("Forecast: STABLE");
    }

    // Outer border frame
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(2000); // DHT22 timing window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect BMP180 and OLED to **GP4/GP5** in parallel. Connect DHT22 to **GP12**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the weather sliders on canvas and observe the OLED weather station update.

## Expected Output

Terminal:
```
Simulation active. Outdoor meteorology engine online.
```

## Expected Canvas Behavior
* Header: White banner reading `METEO STATION HUD` in black text.
* Row 1: `T: 24.5C | H: 50.0%`
* Row 2: `Pres: 1013 hPa`
* Row 3: `Forecast: STABLE`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pressure < 100900` | Analyzes barometric pressure to predict rain or storms when low-pressure systems arrive. |

## Hardware & Safety Concept: Barometric Pressure Forecasts
Rapidly falling pressure indicates incoming low-pressure systems, which bring clouds, wind, and rain. Rising pressure indicates high-pressure systems, bringing clear and sunny conditions. Real weather stations display this **pressure tendency** as trend arrows or weather icons (rain cloud, sun) to make forecasts easy to read at a glance.

## Try This! (Challenges)
1. **Freeze Indicator**: Flash a warning LED on GP15 if the temperature drops below 3.0°C to warn of potential frost.
2. **Altitude Calculator**: Add code to toggle the bottom row to show calculated relative altitude when a button on GP16 is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "BMP180 Sensor Error!" | I2C address conflict | Verify the BMP180 sensor is powered and that SDA/SCL lines are shared correctly with the OLED. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [74 - Pico DHT OLED](../intermediate/74-pico-dht-oled.md)
- [80 - Pico BMP180 Trend](../intermediate/80-pico-bmp180-trend.md)
- [119 - Pico BMP180 OLED](../intermediate/119-pico-bmp180-oled.md)
