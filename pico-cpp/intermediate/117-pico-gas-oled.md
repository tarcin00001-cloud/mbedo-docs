# 117 - Pico Gas OLED

Build a gas concentration monitor that displays MQ-2 smoke levels as a live graph on an OLED screen.

## Goal
Learn how to read analog gas sensors, scale PPM values, and draw dynamic graphic gauges on SSD1306 OLED displays.

## What You Will Build
A gas level safety visualizer:
- **MQ-2 Gas Sensor (GP26)**: Measures analog gas/smoke concentrations.
- **SSD1306 OLED (GP4, GP5)**: Displays the raw concentration level and draws a horizontal bar graph showing the safety threshold boundary.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Heater power |
| MQ-2 Sensor | AO (Analog) | GP26 | Gas level input |
| MQ-2 Sensor | GND | GND | Ground reference |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int MQ2_PIN = 26;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Gas alarm limit (raw analog reading)
const int GAS_LIMIT = 1800;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(MQ2_PIN, INPUT);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int gasLevel = analogRead(MQ2_PIN);

  // Map 12-bit input (0-4095) to screen width (0-128 pixels)
  int barWidth = gasLevel * 128 / 4095;

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("GAS MONITOR");

  // Print raw values
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(10, 20);
  display.print("Level: ");
  display.print(gasLevel);

  // Status message
  display.setCursor(10, 32);
  if (gasLevel > GAS_LIMIT) {
    display.print("STATUS: !!! LEAK !!!");
  } else {
    display.print("STATUS: SAFE");
  }

  // Draw bar graph border
  display.drawRect(0, 46, 128, 14, SSD1306_WHITE);

  // Fill bar graph proportionally
  if (barWidth > 0) {
    display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
  }

  // Draw a dotted vertical line showing the limit threshold (approx column 56)
  display.drawFastVLine(56, 46, 14, SSD1306_WHITE);

  display.display();

  delay(300); // 3 updates per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Gas Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect Gas Sensor to **GP26**, OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the Gas PPM slider and watch the OLED gauge and status change.

## Expected Output

Terminal:
```
Simulation active. OLED gas level monitor active.
```

## Expected Canvas Behavior
| Gas PPM Slider level | OLED Status Readout | OLED Bar Graph |
| --- | --- | --- |
| Low (< 1000) | `STATUS: SAFE` | Narrow bar, left of threshold line |
| High (> 1800) | `STATUS: !!! LEAK !!!` | Wide bar, crossing threshold line |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.drawFastVLine(56, 46, 14, ...)` | Draws a vertical threshold line at column index 56 to show the safety boundary on the graph. |

## Hardware & Safety Concept: Threshold Line Visualization
In safety displays (such as gas, temperature, or pressure monitors), drawing a visible threshold line on a bar graph helps users see immediately how close the system is to a limit. If the bar approaches or crosses the threshold line, users can take action before an alarm sounds.

## Try This! (Challenges)
1. **Audio Alarm**: Connect a buzzer on GP14 and sound a siren if the gas level crosses the threshold.
2. **Flash Warning Screen**: Flash the entire OLED display white on gas alerts to draw attention.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Bar graph stays empty or full on boot | Warm-up phase | MQ sensors require a brief warm-up period (1-2 minutes) to reach operating temperature. Ignore initial extreme values. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [43 - Pico MQ-2 Gas Sensor](../../beginner/43-pico-mq2-gas.md)
- [98 - Pico Gas Leak](98-pico-gas-leak.md)
