# 162 - Pico Weather Station Datalogger

Build an outdoor meteorological tracking station that pools DHT22 climate, BMP180 barometric pressure, LDR sun level, and rain sensor datasets to stream clean CSV logs to Serial.

## Goal
Learn how to pool multiple analog and digital inputs, display status summaries on I2C screens, and stream multi-column CSV datasets to the Serial Monitor.

## What You Will Build
An outdoor environmental datalogger console:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Sensor (GP4, GP5)**: Measures barometric pressure.
- **LDR Sensor (GP26)**: Measures sunlight intensity.
- **Rain Sensor (GP27)**: Measures rain levels.
- **16x2 I2C LCD (GP4, GP5)**: Displays key metrics (e.g. Temp/Hum and Rain status).
- **Serial Datalogger**: Streams CSV-formatted lines (e.g. `Temp,Hum,Pres,Sun,Rain`) every 5 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| LDR Photoresistor | `ldr` | Yes | Yes |
| Rain Drop Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistors | `resistor` | Optional | Yes (voltage dividers) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| LDR Sensor | AO (Signal) | GP26 | Light sensor voltage divider |
| Rain Sensor | AO (Signal) | GP27 | Rain sensor voltage divider |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN  = 12;
const int LDR_PIN  = 26;
const int RAIN_PIN = 27;

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
  pinMode(LDR_PIN, INPUT);
  pinMode(RAIN_PIN, INPUT);

  lcd.init();
  lcd.backlight();

  if (bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Meteo Station");
    lcd.setCursor(0, 1);
    lcd.print("Logging Active  ");

    // Print CSV Header to Serial Monitor
    Serial.println("Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("BMP180 Error!   ");
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
    // 1. Update LCD Screen
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("T:");
    lcd.print(temp, 1);
    lcd.print("C H:");
    lcd.print(humid, 0);
    lcd.print("%");

    lcd.setCursor(0, 1);
    lcd.print("Sun:");
    lcd.print(light);
    lcd.print(" Rain:");
    if (rain < 1500) {
      lcd.print("WET");
    } else {
      lcd.print("DRY");
    }

    // 2. Stream CSV Logs to Serial
    Serial.print(temp, 2);
    Serial.print(",");
    Serial.print(humid, 2);
    Serial.print(",");
    Serial.print(pressure / 100);
    Serial.print(",");
    Serial.print(light);
    Serial.print(",");
    Serial.println(rain);
  }

  delay(5000); // Log once every 5 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, **LDR**, **Rain Sensor** (potentiometer), and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12**, BMP180 and LCD to **GP4/GP5** in parallel, LDR to **GP26**, and Rain Sensor to **GP27**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the sensor sliders and watch the CSV logs update in the Serial terminal.

## Expected Output

Terminal:
```
Temp_C,Hum_pct,Pres_hPa,Light_idx,Rain_idx
24.50,50.20,1013,800,4095
24.60,50.30,1013,810,1200
```

## Expected Canvas Behavior
* Normal state (Dry): LCD reads `T:24.5C H:50%` / `Sun:800 Rain:DRY`.
* Rain detected (Rain sensor slider < 1500): LCD updates to `Rain:WET`, CSV log logs rain level.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial.print(light)` | Outputs the analog sunlight light value to the CSV datalogger stream. |

## Hardware & Safety Concept: CSV Formatting for Data Logging
CSV (Comma-Separated Values) is a standard format for datalogging. Streaming sensor values separated by commas with a carriage return at the end allows software on a computer (like Excel, MATLAB, or Python scripts) to capture the serial stream and save it directly to a spreadsheet for analysis.

## Try This! (Challenges)
1. **Rain Alert Beeper**: Connect a buzzer on GP14 and sound a short beep when rain is first detected.
2. **Relay Fan Actuator**: Connect a relay on GP10 and turn ON an exhaust fan if temperature exceeds 35°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Serial Monitor shows garbled text | Baud rate mismatch | Ensure the Serial Monitor window is configured to 9600 baud rate to match the `Serial.begin(9600)` code settings. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [79 - Pico BMP180 LCD](../intermediate/79-pico-bmp180-lcd.md)
- [123 - Pico Weather Station](123-pico-weather-station.md)
- [129 - Pico Weather Logger](129-pico-weather-logger.md)
