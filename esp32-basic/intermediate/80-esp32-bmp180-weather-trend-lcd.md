# 80 - ESP32 BMP180 Weather Trend LCD

Predict weather trends by monitoring changes in barometric pressure over time and displaying forecasts on a 16x2 I2C LCD screen.

## Goal
Learn how to analyze pressure trends (rising, falling, or steady) to implement basic barometric weather forecasting algorithms.

## What You Will Build
A BMP180 sensor reads atmospheric pressure. The ESP32 compares the current pressure against historical readings. The 16x2 I2C LCD displays the live pressure in hPa on Row 1 and the predicted weather trend (e.g., "STORM ALERT", "FAIR WEATHER", "STABLE") on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| BMP180 Barometric Pressure Sensor | `bmp180` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3V3 | Red | Power |
| BMP180 Sensor | GND | GND | Black | Ground |
| BMP180 Sensor | SDA | GPIO21 | Blue | I2C Data (shared) |
| BMP180 Sensor | SCL | GPIO22 | Yellow | I2C Clock (shared) |
| I2C LCD | VCC | 5V (Vin) | Red | LCD Power |
| I2C LCD | GND | GND | Black | Ground |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data (shared) |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock (shared) |

> **Wiring tip:** The BMP180 and I2C LCD share the I2C bus pins. Connect the SDA and SCL pins in parallel to GPIO 21 and 22.

## Code
```cpp
// BMP180 Weather Trend LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_BMP085.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// History buffer parameters
const int SAMPLE_INTERVAL_S = 5; // Read once every 5 seconds for simulation speed
float lastPressure = 0.0;
unsigned long lastSampleTime = 0;

// Trend change threshold in hPa
const float TREND_THRESHOLD = 0.20; 

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  
  if (!bmp.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("Sensor error!  ");
    while (1) {}
  }
  
  delay(1500);
  lcd.clear();
  
  // Set initial baseline reading
  lastPressure = bmp.readPressure() / 100.0;
  lastSampleTime = millis();
}

void loop() {
  float currentPressure = bmp.readPressure() / 100.0; // In hPa
  unsigned long now = millis();
  
  // Display current pressure on Row 1
  lcd.setCursor(0, 0);
  lcd.print("Pres: ");
  lcd.print(currentPressure, 1);
  lcd.print(" hPa   ");
  
  // Check trend once every interval
  if (now - lastSampleTime >= (SAMPLE_INTERVAL_S * 1000)) {
    float diff = currentPressure - lastPressure;
    
    lcd.setCursor(0, 1);
    if (diff > TREND_THRESHOLD) {
      lcd.print("Trend: Improving"); // Rising pressure
      Serial.println("Trend: Improving (Fair)");
    } 
    else if (diff < -TREND_THRESHOLD) {
      lcd.print("Trend: Stormy   "); // Falling pressure
      Serial.println("Trend: Stormy (Rain)");
    } 
    else {
      lcd.print("Trend: Stable   "); // No significant change
      Serial.println("Trend: Stable");
    }
    
    // Save current as historical baseline for next check
    lastPressure = currentPressure;
    lastSampleTime = now;
  }
  
  delay(100); // Fast loop for LCD rendering responsiveness
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **BMP180 Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Wire SDA to **GPIO21** and SCL to **GPIO22** for both components.
3. Paste the code and click **Run**.
4. Slide the pressure slider down quickly on the BMP180 widget. Wait 5 seconds, and observe the "Stormy" forecast appear on the LCD.

## Expected Output
Serial Monitor:
```
Trend: Stable
Trend: Stormy (Rain)
Trend: Improving (Fair)
```

LCD Display (after dropping pressure):
```
Pres: 1005.4 hPa
Trend: Stormy
```

## Expected Canvas Behavior
* The LCD displays the current pressure in hPa.
* If the simulated pressure remains steady, Row 2 displays "Trend: Stable".
* If the user slides the pressure up or down, the trend label changes accordingly after the next 5-second evaluation interval.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `currentPressure = bmp.readPressure() / 100.0` | Converts raw Pascals output to standard hectopascals (hPa). |
| `diff = currentPressure - lastPressure` | Measures the direction and magnitude of barometric pressure change. |
| `diff < -TREND_THRESHOLD` | Detects a falling pressure trend (threshold offset). |

## Hardware & Safety Concept: Barometric Pressure and Weather Changes
High-pressure zones generally bring dry, clear, stable air (fair weather). Low-pressure zones bring moist, rising air, creating clouds and precipitation (stormy weather). A sudden, rapid drop in barometric pressure indicates a low-pressure front is approaching, warning sailors or weather stations of imminent stormy conditions.

## Try This! (Challenges)
1. **Dynamic Alert Buzzer**: Sound an active buzzer if the pressure drops faster than 0.5 hPa in a single interval.
2. **Larger Buffer**: Expand the trend monitoring window to look at changes over a 10-sample rolling average.
3. **Graphing Trends**: Display a "+" or "-" trend character on Row 1 alongside the live pressure reading.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Trend display does not change | Evaluation interval hasn't elapsed | Wait for the `SAMPLE_INTERVAL_S` window to pass before adjusting slider again |
| Trend fluctuates too easily | `TREND_THRESHOLD` is set too low | Increase the value of `TREND_THRESHOLD` to reduce noise sensitivity |
| LCD output is garbled | Sharing I2C addresses | Ensure other I2C peripherals are not using conflicting addresses |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [78 - ESP32 BMP180 Pressure Sensor Serial Logs](78-esp32-bmp180-pressure-sensor-serial-logs.md)
- [79 - ESP32 BMP180 Temp & Altitude Display](79-esp32-bmp180-temp-altitude-display.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
