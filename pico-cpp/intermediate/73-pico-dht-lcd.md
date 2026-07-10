# 73 - Pico DHT LCD

Display live temperature and humidity readings from a DHT22 sensor on a 16x2 I2C LCD screen.

## Goal
Learn how to interface single-wire climate sensors and format floating-point data values on character displays using I2C.

## What You Will Build
A digital weather display panel:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **16x2 I2C LCD (GP4, GP5)**: Displays the values (e.g. "Temp: 24.5C" on row 0, "Humid: 55.2%" on row 1).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 3.3V | Power supply |
| DHT22 | SDA | GP12 | Data signal (requires 10k pull-up to 3.3V) |
| DHT22 | GND | GND | Ground reference |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  delay(1000);
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();

  if (!isnan(temp) && !isnan(humid)) {
    lcd.clear();
    
    // Row 0: Temperature
    lcd.setCursor(0, 0);
    lcd.print("Temp : ");
    lcd.print(temp);
    lcd.print(" C");

    // Row 1: Humidity
    lcd.setCursor(0, 1);
    lcd.print("Humid: ");
    lcd.print(humid);
    lcd.print(" %");
  }

  delay(2000); // Wait 2 seconds for next reading
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, and **I2C LCD** onto the canvas.
2. Connect DHT22 to **GP12** (with 10k pull-up to 3V3), LCD to **GP4/GP5** (SDA/SCL), and connect VCC/GND.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the DHT22 sliders on canvas and observe the LCD update.

## Expected Output

Terminal:
```
Simulation active. LCD rendering DHT22 data.
```

## Expected Canvas Behavior
* Row 0: `Temp : 24.50 C`
* Row 1: `Humid: 55.20 %`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `lcd.clear()` | Erases all character blocks on the display and resets the cursor to (0, 0). |

## Hardware & Safety Concept: Display Refresh Rates
Clearing and rewriting the entire LCD screen frequently (e.g. every 50 ms) causes the characters to appear dim and flicker, as the liquid crystal fluid takes time to respond. Always limit display refreshes to when the measured variables actually change, or use slow polling intervals (like `delay(2000)` required by the DHT22) to keep characters crisp and legible.

## Try This! (Challenges)
1. **Dynamic Custom Symbols**: Add the custom degree symbol (smiley face or custom CGRAM circle) to display `°` instead of `C`.
2. **Alert Indicator**: Flash an LED on GP15 if the humidity exceeds 80%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes on and off constantly | Too many clears in loop | Make sure `lcd.clear()` is only called inside the slow 2-second loop phase, not in rapid update paths. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [71 - Pico DHT Serial](71-pico-dht-serial.md)
