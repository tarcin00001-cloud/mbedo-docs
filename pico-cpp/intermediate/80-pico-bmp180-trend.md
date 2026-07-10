# 80 - Pico BMP180 Trend

Predict weather trends by monitoring barometric pressure changes over time.

## Goal
Learn how to store historical sensor readings and implement simple trend prediction logic in software.

## What You Will Build
A digital weather forecaster:
- **BMP180 Sensor (GP4 SDA, GP5 SCL)**: Monitors atmospheric pressure.
- **16x2 I2C LCD**: Displays current pressure and predicted trend ("STABLE", "RAIN TREND", or "SUNNY TREND") based on pressure changes.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 | VCC | 3.3V | Power supply |
| BMP180 | SDA | GP4 | Shared I2C Data |
| BMP180 | SCL | GP5 | Shared I2C Clock |
| BMP180 | GND | GND | Ground return |
| I2C LCD | VCC | 5V | LCD Power |
| I2C LCD | SDA | GP4 | Shared I2C Data |
| I2C LCD | SCL | GP5 | Shared I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

long initialPressure = 0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  lcd.init();
  lcd.backlight();
  
  if (bmp.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Calibrating...");
    delay(2000);
    initialPressure = bmp.readPressure(); // Save baseline pressure
  } else {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!");
  }
}

void loop() {
  if (initialPressure > 0) {
    long currentPressure = bmp.readPressure();
    long delta = currentPressure - initialPressure;

    lcd.clear();
    
    // Row 0: Pressure
    lcd.setCursor(0, 0);
    lcd.print("P: ");
    lcd.print(currentPressure / 100); // Display in hPa
    lcd.print(" hPa");

    // Row 1: Trend prediction
    lcd.setCursor(0, 1);
    if (delta < -100) {
      lcd.print("Trend: RAIN/STORM");
    } else if (delta > 100) {
      lcd.print("Trend: SUNNY/DRY");
    } else {
      lcd.print("Trend: STABLE");
    }
  }

  delay(3000); // Check trend every 3 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, and **I2C LCD** onto the canvas.
2. Connect both BMP180 and LCD to GP4 (SDA) and GP5 (SCL) in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the pressure slider down to simulate incoming bad weather, and watch the LCD trend forecast update.

## Expected Output

Terminal:
```
Simulation active. Weather trend forecast engine running.
```

## Expected Canvas Behavior
* Startup: Display shows `Calibrating...` for 2 seconds.
* Standard state: Row 1 displays `Trend: STABLE`.
* Sliding pressure down: Row 1 updates to `Trend: RAIN/STORM`.
* Sliding pressure up: Row 1 updates to `Trend: SUNNY/DRY`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `currentPressure - initialPressure` | Computes the pressure change (delta) since system startup to detect rising or falling trends. |

## Hardware & Safety Concept: Barometric Weather Forecasting
Rapidly falling barometric pressure (delta < -100 Pa) indicates an incoming low-pressure system, which brings clouds, wind, and rain. Conversely, rising pressure (delta > 100 Pa) indicates a high-pressure system, bringing clear, dry, and sunny conditions. Real meteorological systems track pressure changes over 3-hour windows (pressure tendency) to forecast storm events.

## Try This! (Challenges)
1. **Dynamic Calibration**: Connect a button to GP15. Clicking it resets the `initialPressure` baseline to the current reading.
2. **Audio Warning**: Sound a buzzer on GP10 if a rapid drop in pressure (potential storm) is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays stuck on "Calibrating..." | BMP180 initialization failed | Verify that the sensor is receiving power and that I2C connections on GP4/GP5 are correct. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [78 - Pico BMP180 Serial](78-pico-bmp180-serial.md)
- [79 - Pico BMP180 LCD](79-pico-bmp180-lcd.md)
