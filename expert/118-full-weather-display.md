# 118 - Full Weather Display

Combine a DHT22 temperature/humidity sensor and a BMP180 barometric pressure sensor to print detailed weather diagnostics onto an I2C OLED display screen.

## Goal
Learn how to control a high-resolution I2C OLED display, read multiple environmental sensors, and arrange structured multi-line text output on a graphics screen.

## What You Will Build
An environmental display dashboard:
- **Temperature (DHT22)**: Displayed in Celsius.
- **Humidity (DHT22)**: Displayed as a percentage.
- **Barometric Pressure (BMP180)**: Displayed in hectopascals (hPa).
- **Altitude (BMP180)**: Displayed in meters.
- **OLED Graphics Display**: Updates all four parameters in a neat list every 2 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Digital data |
| DHT22 Sensor | GND | GND | Ground reference |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Data bus |
| BMP180 Sensor | SCL | A5 | I2C Clock bus |
| BMP180 Sensor | GND | GND | Ground reference |
| OLED Display | VCC | 5V | Power supply |
| OLED Display | SDA | A4 | I2C Data bus (shared) |
| OLED Display | SCL | A5 | I2C Clock bus (shared) |
| OLED Display | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;
DHT dht(DHT_PIN, DHT_TYPE);

Adafruit_BMP085 bmp;

void setup() {
  Serial.begin(9600);
  dht.begin();
  
  if (!bmp.begin()) {
    Serial.println("BMP085 failed!");
  }

  // Initialize OLED display (address 0x3C or 0x3D)
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("SSD1306 failed!");
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.println("Weather Station");
  display.println("Initializing...");
  display.display();
  delay(2000);
}

void loop() {
  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();
  float pressure = bmp.readPressure() / 100.0; // Pa to hPa
  float altitude = bmp.readAltitude();

  // Print to Serial Monitor
  Serial.print("Temp: ");    Serial.print(tempC);     Serial.print("C | ");
  Serial.print("Hum: ");     Serial.print(humidity);  Serial.print("% | ");
  Serial.print("Press: ");   Serial.print(pressure);  Serial.println("hPa");

  // Print to OLED
  display.clearDisplay();
  display.setCursor(0, 0);
  
  display.setTextSize(1);
  display.println("--- WEATHER HUD ---");
  display.println("");

  display.print("Temp:     ");
  display.print(tempC, 1);
  display.println(" C");

  display.print("Humidity: ");
  display.print(humidity, 1);
  display.println(" %");

  display.print("Pressure: ");
  display.print(pressure, 0);
  display.println(" hPa");

  display.print("Altitude: ");
  display.print(altitude, 0);
  display.println(" m");

  display.display();

  delay(2000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **BMP180 Sensor**, and **OLED Display** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **SDA** to **D2**, **GND** to **GND**.
3. Connect BMP180: **VCC** to **3.3V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
4. Connect OLED: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor, choose interpreted mode, and click **Run**.
6. Modify sliders on the simulated sensors to watch the OLED update.

## Expected Output

Terminal:
```
Temp: 24.00C | Hum: 60.00% | Press: 1013.25hPa
```

OLED Screen:
```
--- WEATHER HUD ---

Temp:     24.0 C
Humidity: 60.0 %
Pressure: 1013 hPa
Altitude: 101 m
```

## Expected Canvas Behavior
| Temp (DHT22) | Hum (DHT22) | Press (BMP180) | OLED Line 2 | OLED Line 3 | OLED Line 4 |
| --- | --- | --- | --- | --- | --- |
| 25.0 C | 50.0% | 1015 hPa | `Temp:     25.0 C` | `Humidity: 50.0 %` | `Pressure: 1015 hPa` |
| -5.2 C | 92.5% | 985 hPa | `Temp:     -5.2 C` | `Humidity: 92.5 %` | `Pressure: 985 hPa` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.clearDisplay()` | Flushes the graphic buffer in RAM, clearing old pixels. |
| `display.display()` | Transfers the graphic RAM buffer over I2C to the physical OLED controller. |
| `tempC, 1` | Limits the float value to one decimal place when printing to keep the UI clean. |

## Hardware & Safety Concept: OLED Buffer Size
Monochrome OLEDs like the SSD1306 are graphics displays. Because the MCU cannot write individual pixels directly over I2C fast enough, the Adafruit library allocates a screen buffer in the Arduino's RAM (1024 bytes for a 128x64 resolution). Since the Arduino Uno has only 2048 bytes of RAM, using an OLED display consumes exactly half of the available memory, which requires memory-efficient coding.

## Try This! (Challenges)
1. **Fahrenheit Toggle**: Add a slide switch or pushbutton to pin D3. When closed, display temperature in Fahrenheit instead of Celsius.
2. **Visual Bar Graph**: Render a simple horizontal bar on the bottom of the screen to indicate the humidity level graphically (e.g. print dashes or draw lines).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen doesn't light up | Wrong I2C address | Change `0x3C` in `display.begin()` to `0x3D`. |
| Out of memory warning | Dynamic RAM overflow | Minimize global variables or use `F()` macro for static strings if compiling. |

## Mode Notes
The custom shims render basic text and values on OLED and are compatible with MbedO interpreted mode.

## Related Projects
- [52 - Humidity Display OLED](../intermediate/52-humidity-display-oled.md)
- [61 - Weather Readout LCD](../intermediate/61-weather-readout-lcd.md)
- [86 - Weather Station](../advanced/86-weather-station.md)
