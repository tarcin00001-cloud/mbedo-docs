# 80 - BMP180 Weather Trend LCD

Predict weather trends by monitoring changes in barometric pressure over time and displaying forecasts on a 16x2 I2C LCD screen.

## Goal
Learn how to analyze pressure trends (rising, falling, or steady) to implement basic barometric weather forecasting algorithms without using loops inside C++ code.

## What You Will Build
A BMP180 sensor reads atmospheric pressure. The board compares the current pressure against historical readings. The 16x2 I2C LCD displays the live pressure in hPa on Row 1 and the predicted weather trend (e.g., "Trend: Stormy", "Trend: Fair", "Trend: Stable") on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Power input (3.3V) |
| BMP180 Sensor | GND | GND | Black | Ground reference |
| BMP180 Sensor | SDA | SDA0 (GP17) | Blue | I2C Data line (shared) |
| BMP180 Sensor | SCL | SCL0 (GP16) | Yellow | I2C Clock line (shared) |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line (shared) |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line (shared) |

> **Wiring tip:** Connect the SDA and SCL pins of the BMP180 and LCD in parallel to SDA0 (GP17) and SCL0 (GP16) on the breadboard.

## Code
```cpp
// BMP180 Weather Trend LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// History tracking variables
float lastPressure = 0.0;
unsigned long lastSampleTime = 0;

// Trend change threshold in hPa
const float TREND_THRESHOLD = 0.20;
bool sensorOk = true;

void setup() {
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("Sensor error!  ");
    sensorOk = false;
  } else {
    // Set initial baseline readings
    lastPressure = bmp.readPressure() / 100.0;
    lastSampleTime = millis();
    lcd.setCursor(0, 1);
    lcd.print("Calibrating... ");
    delay(1500);
    lcd.clear();
  }
}

void loop() {
  if (!sensorOk) {
    delay(2000);
    return;
  }
  
  float currentPressure = bmp.readPressure() / 100.0; // In hPa
  unsigned long now = millis();
  
  // Display current pressure on Row 1
  lcd.setCursor(0, 0);
  lcd.print("Pres: ");
  lcd.print(currentPressure, 1);
  lcd.print(" hPa   ");
  
  // Check pressure trend once every 5 seconds (simulation interval)
  if (now - lastSampleTime >= 5000) {
    float diff = currentPressure - lastPressure;
    
    lcd.setCursor(0, 1);
    if (diff > TREND_THRESHOLD) {
      lcd.print("Trend: Fair     "); // Rising pressure
    } 
    else if (diff < -TREND_THRESHOLD) {
      lcd.print("Trend: Stormy   "); // Falling pressure
    } 
    else {
      lcd.print("Trend: Stable   "); // Stable pressure
    }
    
    // Update baseline
    lastPressure = currentPressure;
    lastSampleTime = now;
  }
  
  delay(100); // Small delay to preserve simulation responsiveness
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **BMP180 Sensor**, and **I2C LCD Display** components onto the canvas.
2. Wire the SDA pins to **SDA0 (GP17)** and SCL pins to **SCL0 (GP16)**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. Slide the pressure slider down on the BMP180 widget. Wait 5 seconds and watch "Trend: Stormy" print on the screen.

## Expected Output
LCD Screen:
```
Pres: 1005.4 hPa
Trend: Stormy
```

## Expected Canvas Behavior
* Row 1 displays the live barometric pressure in hPa.
* If the user shifts the pressure slider up or down on the BMP180 widget, the trend prediction changes to "Fair" or "Stormy" respectively on the next 5-second interval.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `lastPressure = bmp.readPressure() / 100.0` | Stores baseline reading at startup. |
| `now - lastSampleTime >= 5000` | Limits evaluation of weather trends to a 5-second sampling window. |
| `currentPressure - lastPressure` | Computes the magnitude of pressure change. |
| `diff < -TREND_THRESHOLD` | Evaluates if the pressure has dropped past the threshold. |

## Hardware & Safety Concept
* **Barometric Weather Systems**: High-pressure zones generally bring dry, clear, stable air (fair weather). Low-pressure zones bring moist, rising air, creating clouds and precipitation (stormy weather). A sudden, rapid drop in barometric pressure indicates a low-pressure front is approaching, warning sailors or weather stations of imminent stormy conditions.

## Try This! (Challenges)
1. **Storm Alarm**: Add an active buzzer on GPIO 14 that sounds a warning pattern if a "Stormy" trend is detected.
2. **Dynamic Trend Indicator**: Print a visual arrow symbol (`->` for stable, `^` for rising, `v` for falling) on Row 1 next to the pressure reading.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Forecast doesn't change | 5-second interval has not elapsed | Wait 5 seconds after adjusting the slider before expecting updates. |
| Trend constantly fluctuates | Threshold is too low | Increase the value of `TREND_THRESHOLD` in the code. |
| LCD output is garbled | Shared I2C address conflict | Check the I2C backpack configuration for address conflicts. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [78 - BMP180 Pressure Sensor Serial Logs](78-bmp180-pressure-sensor-serial.md)
- [79 - BMP180 Temp & Altitude Display](79-bmp180-temp-altitude-display.md)
- [73 - DHT22 Temperature LCD HUD](73-dht22-temperature-lcd-hud.md)
