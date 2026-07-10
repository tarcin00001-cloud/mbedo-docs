# 74 - Pico DHT OLED

Display live climate readings from a DHT22 sensor on a 0.96-inch SSD1306 OLED screen.

## Goal
Learn how to interface single-wire digital sensors and design graphic display layouts to align text labels on OLED screens.

## What You Will Build
A digital weather console HUD:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **SSD1306 OLED (GP4, GP5)**: Displays the values with structured text alignment (e.g. "TEMP: 24.5 C" and "HUM: 55.2%").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 3.3V | Power supply |
| DHT22 | SDA | GP12 | Data signal (requires 10k pull-up to 3.3V) |
| DHT22 | GND | GND | Ground reference |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN = 12;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

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
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 10);
    display.println("Initializing...");
    display.display();
    delay(1000);
  }
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(20, 3);
    display.print("WEATHER STATION");

    // Draw readings
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(10, 26);
    display.print("Temp : ");
    display.print(temp);
    display.print(" C");

    display.setCursor(10, 42);
    display.print("Humid: ");
    display.print(humid);
    display.print(" %");

    // Draw frame border
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);

    display.display();
  }

  delay(2000); // DHT22 rate limit
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12** (with 10k pull-up to 3V3), OLED to **GP4/GP5** (SDA/SCL), and connect VCC/GND.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the DHT22 sliders on canvas and observe the OLED update.

## Expected Output

Terminal:
```
Simulation active. OLED rendering weather station HUD.
```

## Expected Canvas Behavior
* Top: Solid white header bar containing the text `WEATHER STATION` in black.
* Middle: Two lines reading `Temp : 24.50 C` and `Humid: 55.20 %`.
* Border: A white bounding rectangle around the screen edge.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.drawRect(0, 0, 128, 64, SSD1306_WHITE)` | Draws an empty bounding rectangle around the outer edge of the display. |

## Hardware & Safety Concept: I2C Speed Modes
The standard I2C speed is 100 kHz (Standard Mode), but it can be increased to 400 kHz (Fast Mode) or 1 MHz (Fast Mode Plus) depending on the pull-up resistor values. When updating graphics on high-resolution OLED displays, using 400 kHz speeds reduces the buffer transmission time from 100 ms to under 25 ms, allowing smoother animations and faster refresh rates.

## Try This! (Challenges)
1. **Dynamic Progress Bar**: Draw a horizontal battery-style bar graph showing relative humidity level.
2. **Fahrenheit Toggle**: Connect a button to GP15 to switch the display units between Celsius and Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "Initializing..." but never updates | Sensor read failure | Verify the data connection pin GP12. The code skips updating the display if `isnan(temp)` evaluates true. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [61 - Pico OLED HUD](61-pico-oled-hud.md)
- [73 - Pico DHT LCD](73-pico-dht-lcd.md)
