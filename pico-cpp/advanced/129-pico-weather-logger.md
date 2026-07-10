# 129 - Pico Weather Logger

Build a complete weather logger that displays climate telemetry on a 16x2 LCD and streams formatted CSV logs to the Serial Monitor.

## Goal
Learn how to pool climate telemetry from multiple I2C and single-wire digital sensors, format data streams, and update I2C text screens.

## What You Will Build
An environmental datalogger console:
- **BMP180 Sensor (GP4, GP5)**: Measures barometric pressure.
- **DHT22 Sensor (GP12)**: Measures temperature and relative humidity.
- **16x2 I2C LCD (GP4, GP5)**: Displays live temperature and humidity values.
- **Serial Monitor**: Streams CSV-formatted log lines (e.g. `24.5,50.2,101325`) every 3 seconds for datalogging.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| All Devices | VCC | 3.3V (5V for LCD) | Power supply lines |
| All Devices | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  
  lcd.init();
  lcd.backlight();
  
  if (bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Weather Logger");
    lcd.setCursor(0, 1);
    lcd.print("Online & Logging");
    
    // Print CSV Header to Serial Monitor
    Serial.println("Temp_C,Hum_pct,Pres_Pa");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!");
    while (1);
  }
  delay(1500);
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();
  long pressure = bmp.readPressure();

  if (!isnan(temp) && !isnan(humid)) {
    // 1. Update LCD Screen
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T: ");
    lcd.print(temp, 1);
    lcd.print("C  H: ");
    lcd.print(humid, 0);
    lcd.print("%");

    lcd.setCursor(0, 1);
    lcd.print("P: ");
    lcd.print(pressure / 100);
    lcd.print(" hPa");

    // 2. Stream CSV Logs to Serial
    Serial.print(temp, 2);
    Serial.print(",");
    Serial.print(humid, 2);
    Serial.print(",");
    Serial.println(pressure);
  }

  delay(3000); // Log once every 3 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, **DHT22**, and **I2C LCD** onto the canvas.
2. Connect BMP180 and LCD to **GP4/GP5** in parallel. Connect DHT22 to **GP12**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Observe CSV log coordinates print in the Serial terminal and values display on the LCD.

## Expected Output

Terminal:
```
Temp_C,Hum_pct,Pres_Pa
24.50,50.20,101325
24.60,50.30,101320
```

## Expected Canvas Behavior
* Row 0: `T: 24.5C  H: 50%`
* Row 1: `P: 1013 hPa`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial.println("Temp_C,Hum_pct,Pres_Pa")` | Prints the CSV header line at startup, allowing datalogging software (like Excel) to import the data correctly. |

## Hardware & Safety Concept: CSV Formatting for Data Logging
CSV (Comma-Separated Values) is a standard format for datalogging. Streaming sensor values separated by commas with a carriage return at the end allows software on a computer (like Excel, MATLAB, or Python scripts) to capture the serial stream and save it directly to a spreadsheet for analysis.

## Try This! (Challenges)
1. **Datalogging Indicator LED**: Flash an LED on GP13 for 100 ms every time a new CSV log line is transmitted.
2. **Alert Trigger**: Sound a buzzer on GP14 if the temperature exceeds 35°C or pressure drops rapidly.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor shows garbled text | Baud rate mismatch | Ensure the Serial Monitor window is configured to 9600 baud rate to match the `Serial.begin(9600)` code settings. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [73 - Pico DHT LCD](../intermediate/73-pico-dht-lcd.md)
- [79 - Pico BMP180 LCD](../intermediate/79-pico-bmp180-lcd.md)
- [123 - Pico Weather Station](123-pico-weather-station.md)
