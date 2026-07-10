# 139 - Pico DHT OLED Graph

Build a climate monitoring station that tracks and displays current, minimum, and maximum temperature limits on an OLED screen.

## Goal
Learn how to track historical extrema (minimum/maximum) using memory variables (no arrays) and draw structured comparison layouts on SSD1306 OLED displays.

## What You Will Build
A climate statistics dashboard:
- **DHT22 Sensor (GP12)**: Measures ambient temperature.
- **SSD1306 OLED (GP4, GP5)**: Displays the current temperature along with the historical minimum and maximum temperatures recorded since startup.

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
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

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

float minTemp = 100.0;
float maxTemp = -100.0;
bool firstReading = true;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  float temp = dht.readTemperature();

  if (!isnan(temp)) {
    // Initialize limits on first valid reading
    if (firstReading) {
      minTemp = temp;
      maxTemp = temp;
      firstReading = false;
    }

    // Update historical limits
    if (temp < minTemp) { minTemp = temp; }
    if (temp > maxTemp) { maxTemp = temp; }

    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(15, 3);
    display.print("TEMP STATS ENGINE");

    // Display values
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Current Temp display
    display.setCursor(10, 20);
    display.print("Current: ");
    display.print(temp, 1);
    display.print(" C");

    // Min/Max bounds display
    display.setCursor(10, 34);
    display.print("Min Temp: ");
    display.print(minTemp, 1);
    display.print(" C");

    display.setCursor(10, 48);
    display.print("Max Temp: ");
    display.print(maxTemp, 1);
    display.print(" C");

    // Outer border frame
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(2000); // DHT22 timing window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12** (with 10k pull-up), OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature slider on the DHT22 model and watch the Current, Min, and Max limits update on the OLED.

## Expected Output

Terminal:
```
Simulation active. Temperature tracking engine online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Current: 24.0 C`, `Min Temp: 24.0 C`, `Max Temp: 24.0 C` (on first reading).
* Slide temperature up to 30°C: `Current` and `Max Temp` update to 30.0°C.
* Slide temperature down to 18°C: `Current` updates to 18.0°C, `Min Temp` updates to 18.0°C, while `Max Temp` remains at 30.0°C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp < minTemp` | Updates the historical minimum temperature variable if the current reading drops below the previous minimum. |

## Hardware & Safety Concept: Limit Tracking and Extreme Values
Tracking extreme values (min/max temperature limits) is critical in warehouse climate control, server rooms, and vaccine storage. If the temperature exceeds safe limits even briefly, it can damage products. Dataloggers track these extrema to verify that the storage environment stayed within safe boundaries throughout the entire transit process.

## Try This! (Challenges)
1. **Reset Button**: Connect a button on GP15 that resets the min and max temperature trackers to the current reading.
2. **Audio Warning**: Sound a buzzer on GP14 if the temperature exceeds a safety limit of 35°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Min and Max read extreme default values on boot | Initializer check missing | Ensure the code sets `minTemp` and `maxTemp` to the first valid reading (`firstReading`) to prevent them from staying stuck at default placeholder values. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [74 - Pico DHT OLED](../intermediate/74-pico-dht-oled.md)
- [116 - Pico DHT OLED HUD](../intermediate/116-pico-dht-oled-hud.md)
- [123 - Pico Weather Station](123-pico-weather-station.md)
