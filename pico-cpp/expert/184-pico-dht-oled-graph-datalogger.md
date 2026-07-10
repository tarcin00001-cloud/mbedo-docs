# 184 - Pico DHT OLED Graph Datalogger

Build an environmental monitoring station that tracks current, minimum, and maximum temperature limits, displays a vertical bar graph on an OLED, and streams CSV logs over Bluetooth.

## Goal
Learn how to track temperature extremes (minimum/maximum) without arrays, design vertical scale bar graphs on SSD1306 displays, and stream formatted CSV logs over Bluetooth.

## What You Will Build
A climate statistics terminal with wireless logging:
- **DHT22 Sensor (GP12)**: Measures ambient room temperature.
- **SSD1306 OLED (GP4, GP5)**: Displays the current temperature, records historical minimum/maximum limits since startup, and draws a vertical temperature scale bar graph.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits CSV logs wirelessly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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
  Serial1.begin(9600); // Bluetooth hardware UART
  dht.begin();

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  // Print CSV Header to Bluetooth
  Serial1.println("Temp_C,Min_C,Max_C");
  delay(1000);
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

    // Update OLED Screen
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(12, 3);
    display.print("TEMP PROFILE LOGGER");

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
    display.drawRect(105, 18, 14, 40, SSD1306_WHITE);
    if (barHeight > 0) {
      display.fillRect(107, 18 + (36 - barHeight) + 2, 10, barHeight - 4, SSD1306_WHITE);
    }

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    // Stream CSV logs over Bluetooth
    Serial1.print(temp, 2);
    Serial1.print(",");
    Serial1.print(minTemp, 2);
    Serial1.print(",");
    Serial1.println(maxTemp, 2);
  }

  delay(2000); // DHT22 rate window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect DHT22 to **GP12**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature on the DHT22, and watch the OLED scale bar and Bluetooth logs update.

## Expected Output

Terminal (Bluetooth):
```
Temp_C,Min_C,Max_C
24.50,24.50,24.50
28.20,24.50,28.20
```

## Expected Canvas Behavior
* Startup: OLED reads `Current: 24.0 C`, `Min: 24.0 C`, `Max: 24.0 C`. Bluetooth streams starting telemetry log.
* Slide temperature to 30°C: OLED updates, scale bar rises, Bluetooth prints `30.00,24.00,30.00`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.print(minTemp, 2)` | Sends the recorded minimum temperature over Bluetooth for remote analysis. |

## Hardware & Safety Concept: Cold Chain Thermal Storage
Vaccines and pharmaceutical products are highly sensitive to temperature variations. Cold chain transport boxes use vacuum-insulated panels and phase-change materials (PCM) to keep temperatures between 2°C and 8°C. Active dataloggers run continuously to record and upload temperature data, verifying that the cargo stayed within safe limits throughout transport.

## Try This! (Challenges)
1. **Reset Button**: Connect a button on GP14 to reset the stored min and max temperature trackers.
2. **Audio Indicator**: Sound a buzzer on GP15 if the temperature exceeds a safety limit of 35°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph drawing overflows the screen borders | Bar offset calculation error | Ensure that the vertical fill coordinate uses the offset height mapping `18 + (36 - barHeight)` to prevent drawing outside display boundaries. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [116 - Pico DHT OLED HUD](../intermediate/116-pico-dht-oled-hud.md)
- [139 - Pico DHT OLED Graph](../advanced/139-pico-dht-oled-graph.md)
- [168 - Pico DHT OLED Datalogger](../advanced/168-pico-dht-oled-datalogger.md)
