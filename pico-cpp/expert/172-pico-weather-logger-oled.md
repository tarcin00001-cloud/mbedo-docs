# 172 - Pico Weather Logger OLED

Build a wireless meteorological station that displays climate telemetry on an OLED screen and streams CSV logs over Bluetooth.

## Goal
Learn how to pool multiple I2C and analog outdoor sensors (DHT22, BMP180, LDR, Rain), format graphic layouts on OLED screens, and stream structured CSV telemetry over Bluetooth.

## What You Will Build
An outdoor weather telemetry console:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Sensor (GP4, GP5)**: Measures barometric air pressure.
- **LDR Sensor (GP26)**: Measures sunlight intensity.
- **Rain Sensor (GP27)**: Measures rain levels.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless weather logs.
- **SSD1306 OLED (GP4, GP5)**: Displays active weather status and charts.
- **CSV Data Stream**: Transmits logs (e.g. `Temp,Hum,Pres,Sun,Rain`) over Bluetooth.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Rain Drop Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage dividers) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| LDR Sensor | AO (Signal) | GP26 | Light sensor divider |
| Rain Sensor | AO (Signal) | GP27 | Rain sensor divider |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN  = 12;
const int LDR_PIN  = 26;
const int RAIN_PIN = 27;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  pinMode(LDR_PIN, INPUT);
  pinMode(RAIN_PIN, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  if (bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("Weather Station");
    display.setCursor(10, 35);
    display.print("Bluetooth Active");
    display.display();
    
    // Print CSV Header over Bluetooth
    Serial1.println("Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx");
  } else {
    display.clearDisplay();
    display.setCursor(10, 20);
    display.print("BMP180 Error!");
    display.display();
    while (1);
  }
  delay(1500);
}

void loop() {
  float temp  = dht.readTemperature();
  float humid = dht.readHumidity();
  long pressure = bmp.readPressure();
  int light   = analogRead(LDR_PIN);
  int rain    = analogRead(RAIN_PIN);

  if (!isnan(temp) && !isnan(humid)) {
    // 1. Update OLED Screen
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(20, 3);
    display.print("METEO TELEMETRY");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Temp & Humidity
    display.setCursor(8, 20);
    display.print("T: ");
    display.print(temp, 1);
    display.print("C | H: ");
    display.print(humid, 0);
    display.print("%");

    // Row 2: Pressure
    display.setCursor(8, 34);
    display.print("Pres: ");
    display.print(pressure / 100);
    display.print(" hPa");

    // Row 3: Light & Rain status
    display.setCursor(8, 48);
    display.print("Sun : ");
    display.print(light);
    display.print(" | ");
    if (rain < 1500) {
      display.print("RAIN");
    } else {
      display.print("DRY");
    }

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    // 2. Stream CSV Logs over Bluetooth
    Serial1.print(temp, 2);
    Serial1.print(",");
    Serial1.print(humid, 2);
    Serial1.print(",");
    Serial1.print(pressure / 100);
    Serial1.print(",");
    Serial1.print(light);
    Serial1.print(",");
    Serial1.println(rain);
  }

  delay(4000); // Update once every 4 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, **LDR**, **Rain Sensor** (potentiometer), **HC-05**, and **SSD1306 OLED** onto the canvas.
2. Connect sensors and communication lines. Connect BMP180/OLED in parallel to I2C lines.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weather sliders on canvas, check the OLED status dashboard, and verify that wireless CSV logs are printing.

## Expected Output

Terminal (Bluetooth):
```
Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx
24.50,50.20,1013,800,4095
24.60,50.30,1013,810,1200
```

## Expected Canvas Behavior
* Startup: OLED shows `METEO TELEMETRY` dashboard.
* Rain slider < 1500: OLED displays `Rain : RAIN`, Bluetooth streams corresponding CSV values.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print(temp, 2)` | Sends the temperature value rounded to two decimal places over the wireless Bluetooth connection. |

## Hardware & Safety Concept: Industrial Telemetry Design
Remote telemetry nodes are exposed to harsh weather conditions. To protect the sensors, boards are mounted inside **Stevenson screens** (louvered enclosures that allow air passage but shield devices from direct rain and solar radiation). Power is typically supplied by solar panels paired with weather-resistant lithium iron phosphate ($LiFePO_4$) batteries.

## Try This! (Challenges)
1. **Critical Low Temp Warning**: Sound a buzzer on GP14 if temperature drops below 3°C (frost warning).
2. **Dynamic Log rate**: Stream CSV records faster (every 1 second) when rain is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen coordinates are distorted | Pixel memory bounds overflow | Ensure all drawing coordinate commands use relative values that stay within the 128x64 display window boundary. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [123 - Pico Weather Station](../advanced/123-pico-weather-station.md)
- [129 - Pico Weather Logger](../advanced/129-pico-weather-logger.md)
- [162 - Pico Weather Station Datalogger](../advanced/162-pico-weather-station-datalogger.md)
