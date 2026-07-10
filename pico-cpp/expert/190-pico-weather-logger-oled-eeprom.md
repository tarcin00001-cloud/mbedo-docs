# 190 - Pico Weather Logger OLED EEPROM

Build a wireless environmental monitoring station that saves altitude references in non-volatile flash memory (EEPROM), updates an OLED, and streams weather logs over Bluetooth.

## Goal
Learn how to interface multiple I2C and digital sensors (DHT22, BMP180), display coordinates on OLED screens, read/write floats to EEPROM, and stream Bluetooth CSV logs.

## What You Will Build
An atmospheric monitoring station:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **BMP180 Sensor (GP4, GP5)**: Measures barometric pressure.
- **EEPROM (Non-volatile Flash)**: Stores local sea-level pressure configurations (default `1013` hPa) at address `50` to calculate accurate relative altitudes.
- **SSD1306 OLED (GP4, GP5)**: Displays live climate telemetry.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless CSV logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Climate Sensor | `dht22` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| BMP180 Sensor | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <EEPROM.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

int seaLevelPres_hPa = 1013; // Default reference

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  EEPROM.begin(512);

  // Read stored sea-level pressure configuration from EEPROM (address 50-51)
  byte b0 = EEPROM.read(50);
  byte b1 = EEPROM.read(51);
  int storedVal = (b0 << 8) | b1;

  if (storedVal >= 900 && storedVal <= 1100) { // Valid pressure limits
    seaLevelPres_hPa = storedVal;
  }

  if (bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("Weather Terminal");
    display.setCursor(10, 35);
    display.print("EEPROM Ref Loaded");
    display.display();
    
    // Print CSV Header to Bluetooth
    Serial1.println("Temp_C,Hum_pct,Pres_hPa,Alt_m");
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
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();
  long pressure = bmp.readPressure(); // Pa

  // Calculate altitude based on stored sea level reference pressure
  float altitude = bmp.readAltitude(seaLevelPres_hPa * 100);

  // Check for configuration commands over Bluetooth
  // Commands: '+' (Increment reference), '-' (Decrement reference)
  if (Serial1.available()) {
    char cmd = Serial1.read();
    if (cmd == '+') {
      seaLevelPres_hPa = seaLevelPres_hPa + 1;
      saveReferenceToEEPROM();
    } else if (cmd == '-') {
      seaLevelPres_hPa = seaLevelPres_hPa - 1;
      saveReferenceToEEPROM();
    }
  }

  if (!isnan(temp) && !isnan(humid)) {
    // 1. Update OLED Screen
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(18, 3);
    display.print("METEO TERMINAL");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Temp & Humidity
    display.setCursor(8, 20);
    display.print("T: ");
    display.print(temp, 1);
    display.print("C | H: ");
    display.print(humid, 0);
    display.print("%");

    // Row 2: Pressure & Alt
    display.setCursor(8, 34);
    display.print("P: ");
    display.print(pressure / 100);
    display.print(" hPa");

    // Row 3: Altitude & Sea Level Ref
    display.setCursor(8, 48);
    display.print("Alt: ");
    display.print(altitude, 1);
    display.print("m (Ref:");
    display.print(seaLevelPres_hPa);
    display.print(")");

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    // 2. Stream CSV Logs over Bluetooth
    Serial1.print(temp, 2);
    Serial1.print(",");
    Serial1.print(humid, 2);
    Serial1.print(",");
    Serial1.print(pressure / 100);
    Serial1.print(",");
    Serial1.println(altitude, 2);
  }

  delay(3000); // 3-second update rate
}

void saveReferenceToEEPROM() {
  byte b0 = (seaLevelPres_hPa >> 8) & 0xFF;
  byte b1 = seaLevelPres_hPa & 0xFF;
  EEPROM.write(50, b0);
  EEPROM.write(51, b1);
  EEPROM.commit();
  Serial1.print("EEPROM updated: Sea-level pressure reference = ");
  Serial1.print(seaLevelPres_hPa);
  Serial1.println(" hPa");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **BMP180**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect sensors and communication lines. Connect OLED/BMP180 to **GP4/GP5** in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `+` or `-` in the Bluetooth terminal to adjust the sea-level reference pressure, and verify that the calculated altitude changes and stays saved.

## Expected Output

Terminal (Bluetooth):
```
Temp_C,Hum_pct,Pres_hPa,Alt_m
24.50,50.20,1013,10.25
EEPROM updated: Sea-level pressure reference = 1014 hPa
24.50,50.20,1013,18.50
```

## Expected Canvas Behavior
* Startup: OLED shows `METEO TERMINAL` showing climate stats and calculated altitude.
* Bluetooth command `+`: OLED reference pressure updates to `1014`, altitude recalculated.
* Restart Simulation: Reference pressure remains at `1014` loaded from memory.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `bmp.readAltitude(seaLevelPres_hPa * 100)` | Calculates the relative altitude by comparing local measured air pressure with the reference sea-level pressure. |

## Hardware & Safety Concept: Altimeter Calibration Persistence
Barometric altimeters calculate altitude relative to sea-level pressure. Since sea-level pressure changes daily with the weather, altimeters must be calibrated frequently. Storing this reference configuration in non-volatile flash memory ensures that the altimeter boots up with the correct calibration settings.

## Try This! (Challenges)
1. **Low Pressure Alert**: Sound a buzzer on GP14 if the pressure drops rapidly (indicating incoming storm conditions).
2. **Display Sleep**: Turn OFF the OLED screen automatically if no keys are pressed for 15 seconds to save battery.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Altitude readings drift over hours | Weather changes | Barometric pressure changes with weather. Re-calibrate reference values daily to ensure accurate altitude readings. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [119 - Pico BMP180 OLED](../intermediate/119-pico-bmp180-oled.md)
- [123 - Pico Weather Station](../advanced/123-pico-weather-station.md)
- [162 - Pico Weather Station Datalogger](../advanced/162-pico-weather-station-datalogger.md)
- [172 - Pico Weather Logger OLED](172-pico-weather-logger-oled.md)
