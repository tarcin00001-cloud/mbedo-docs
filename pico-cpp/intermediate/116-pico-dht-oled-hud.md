# 116 - Pico DHT OLED HUD

Build a high-resolution indoor climate HUD that displays temperature and humidity statistics with graphic borders on an OLED screen.

## Goal
Learn how to display multi-variable environmental telemetry from digital sensors (DHT22) using structured layouts on SSD1306 OLED screens.

## What You Will Build
An indoor climate console:
- **DHT22 Sensor (GP12)**: Measures temperature and humidity.
- **SSD1306 OLED (GP4, GP5)**: Displays a clean dashboard layout containing temperature, humidity, and a comfort indicator message.

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
    display.println("Climate HUD Init...");
    display.display();
    delay(1000);
  }
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(15, 3);
    display.print("CLIMATE DASHBOARD");

    // Display temperature
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("Temp: ");
    display.print(temp, 1);
    display.print(" C");

    // Display humidity
    display.setCursor(10, 32);
    display.print("Hum : ");
    display.print(humid, 1);
    display.print(" %");

    // Comfort indicator zone
    display.setCursor(10, 48);
    if (temp >= 20.0 && temp <= 26.0 && humid >= 30.0 && humid <= 60.0) {
      display.print("Comfort: EXCELLENT");
    } else if (temp > 30.0 || humid > 70.0) {
      display.print("Comfort: HOT/HUMID");
    } else {
      display.print("Comfort: MODERATE");
    }

    // Outer border frame
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(2000); // Wait 2 seconds for next reading
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12** (with 10k pull-up), OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature and humidity sliders on the DHT22 model and observe the dashboard update.

## Expected Output

Terminal:
```
Simulation active. OLED indoor climate HUD online.
```

## Expected Canvas Behavior
* Header: White banner reading `CLIMATE DASHBOARD` in black text.
* Row 1: `Temp: 24.5 C`
* Row 2: `Hum : 50.0 %`
* Row 3: `Comfort: EXCELLENT`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp >= 20.0 && temp <= 26.0` | Checks if ambient conditions are within the optimal indoor comfort range to update the display message. |

## Hardware & Safety Concept: Display Layout Design
When designing graphics for small OLED screens, it is important to align text labels cleanly. Using coordinates (X, Y) to group related data makes the screen easier to read at a glance. Adding borders, division lines, or reverse-color header banners (e.g. black text on a white rectangle block) helps organize the screen layout.

## Try This! (Challenges)
1. **Auditory Warning**: Connect a buzzer on GP14 and sound a chime if comfort levels become poor.
2. **Fahrenheit Toggle**: Connect a button on GP16 to switch temperature displays between Celsius and Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED displays "Climate HUD Init..." and never updates | DHT22 reading error | Check the DHT22 data pin wiring on GP12. The display only redraws if the sensor reports valid temperature and humidity readings. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [61 - Pico OLED HUD](61-pico-oled-hud.md)
- [74 - Pico DHT OLED](74-pico-dht-oled.md)
- [100 - Pico Smart Fan](100-pico-smart-fan.md)
