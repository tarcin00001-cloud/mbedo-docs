# 168 - Pico DHT OLED Datalogger

Build a temperature-tracking station that logs historical extremes and displays a vertical temperature scale bar on an OLED screen.

## Goal
Learn how to track climate changes over time, calculate minimum/maximum extremes without using array memory buffers, and draw relative vertical bar graphs on SSD1306 OLED displays.

## What You Will Build
A temperature profile visualizer:
- **DHT22 Sensor (GP12)**: Measures ambient room temperature.
- **SSD1306 OLED (GP4, GP5)**: Displays the current temperature, records historical minimum/maximum temperature limits since startup, and draws a vertical temperature scale bar graph.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

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

    // Update extremes
    if (temp < minTemp) { minTemp = temp; }
    if (temp > maxTemp) { maxTemp = temp; }

    // Map temperature (0-40 C) to vertical bar height (0-36 pixels)
    int barHeight = temp * 36 / 40;
    if (barHeight < 0) { barHeight = 0; }
    if (barHeight > 36) { barHeight = 36; }

    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(12, 3);
    display.print("TEMP PROFILE LOGGER");

    // Display statistics
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    
    display.setCursor(10, 20);
    display.print("Current: ");
    display.print(temp, 1);
    display.print(" C");

    display.setCursor(10, 32);
    display.print("Min    : ");
    display.print(minTemp, 1);
    display.print(" C");

    display.setCursor(10, 44);
    display.print("Max    : ");
    display.print(maxTemp, 1);
    display.print(" C");

    // Draw vertical temperature scale bar graph on the right side
    display.drawRect(105, 18, 14, 40, SSD1306_WHITE); // Frame
    if (barHeight > 0) {
      display.fillRect(107, 18 + (36 - barHeight) + 2, 10, barHeight - 4, SSD1306_WHITE); // Fill bar
    }

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(2000); // DHT22 rate window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 to **GP12**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature on the DHT22 up and down, and watch the scale bar rise and fall and the minimum/maximum bounds track the changes.

## Expected Output

Terminal:
```
Simulation active. Temperature profile logger online.
```

## Expected Canvas Behavior
* Startup: OLED reads `Current: 24.0 C`, `Min: 24.0 C`, `Max: 24.0 C`. Scale bar fills to ~60% height.
* Slide temperature to 35°C: `Current` and `Max` update to 35.0°C. Scale bar rises.
* Slide temperature to 15°C: `Current` and `Min` update to 15.0°C. Scale bar falls, while `Max` remains at 35.0°C.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp * 36 / 40` | Scales the temperature (0 to 40°C) to fit within the 36-pixel height of the graphical bar on the OLED. |

## Hardware & Safety Concept: Extremum Telemetry Logs
Temperature recorders are critical in medical cold chains (such as vaccine shipping boxes). If temperature levels exceed safe limits even once, the batch is compromised. These devices record extreme values (minimum and maximum) to verify that cargo stayed within safe boundaries throughout transit.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP14 and sound a beep if the temperature exceeds a safety limit of 30°C.
2. **Clear Memory Button**: Connect a button on GP15 that resets the min and max temperature values to the current temperature.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph drawing overflows the screen borders | Bar offset calculation error | Ensure that the vertical fill coordinate uses the offset height mapping `18 + (36 - barHeight)` to prevent drawing outside display boundaries. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [74 - Pico DHT OLED](../intermediate/74-pico-dht-oled.md)
- [116 - Pico DHT OLED HUD](../intermediate/116-pico-dht-oled-hud.md)
- [139 - Pico DHT OLED Graph](139-pico-dht-oled-graph.md)
