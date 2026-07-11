# 74 - DHT22 Temperature OLED Graph

Plot live temperature variations from a DHT22 sensor as a graphical line chart on an SSD1306 OLED screen.

## Goal
Learn how to scale sensor value updates to pixel screen coordinates, and draw lines dynamically to generate graphs without using loops or array buffers.

## What You Will Build
A DHT22 sensor is connected to GPIO 12, and an SSD1306 OLED display is connected via I2C. The OLED screen draws a line graph showing how the temperature changes over time.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| 0.96" SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | VCC (Pin 1) | 3V3 | Red | Power input |
| DHT22 Sensor | DATA (Pin 2) | GPIO 12 | Yellow | Bidirectional data line |
| DHT22 Sensor | GND (Pin 4) | GND | Black | Ground reference |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power supply (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The SSD1306 OLED runs on 3.3V power (do not connect VCC to 5V). SDA and SCL connect to ARIES hardware I2C0 pins GP17 and GP16.

## Code
```cpp
// DHT22 Temperature OLED Graph
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <DHT.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C
#define DHTPIN 12
#define DHTTYPE DHT22

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);
DHT dht(DHTPIN, DHTTYPE);

int currentX = 0;
int lastY = 32;

void setup() {
  Wire.begin();
  display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Temp Graph (10-40C)");
  display.display();
  
  dht.begin();
}

void loop() {
  // Wait 2 seconds between reads
  delay(2000);
  
  float temperature = dht.readTemperature();
  
  if (isnan(temperature)) {
    return; // Ignore invalid reading
  }
  
  // Constrain temperature between 10C and 40C
  float tempConstrained = temperature;
  if (tempConstrained < 10.0) tempConstrained = 10.0;
  if (tempConstrained > 40.0) tempConstrained = 40.0;
  
  // Map temperature (10C to 40C) to Y coordinates (63 down to 16)
  // Higher temperature maps to lower Y coordinate (top of screen)
  int currentY = 63 - (int)((tempConstrained - 10.0) * (63.0 - 16.0) / 30.0);
  
  if (currentX == 0) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.print("T: ");
    display.print(temperature, 1);
    display.print(" *C");
    display.display();
    lastY = currentY;
  } else {
    // Draw line segment from last point to current point
    display.drawLine(currentX - 2, lastY, currentX, currentY, SSD1306_WHITE);
    display.display();
    lastY = currentY;
  }
  
  currentX += 2; // Step 2 pixels horizontally
  if (currentX >= 128) {
    currentX = 0; // Wrap around to start of screen
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DHT22**, and **SSD1306 OLED** components onto the canvas.
2. Connect the DHT22 DATA to **GPIO 12**.
3. Connect the OLED SDA/SCL pins to **SDA0 (GP17)** and **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Change the temperature slider on the DHT22 widget and observe the line graph update on the OLED.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED SSD1306 Initialized at 0x3C.
```

## Expected Canvas Behavior
* At startup, the OLED displays `Temp Graph (10-40C)`.
* Every 2 seconds, a new point is plotted. The screen draws a continuous line showing the temperature trend.
* When the line reaches the right edge, the screen clears and starts drawing from the left side again.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Adafruit_SSD1306 display(...)` | Configures size and references Wire object. |
| `tempConstrained < 10.0` | Bounds reading to fit inside graphic limits. |
| `63 - (int)(...)` | Scales float temperature to screen pixels (Y coordinates). |
| `display.drawLine(...)` | Draws line connections to form a continuous graph. |
| `currentX += 2` | Progresses coordinate position. |

## Hardware & Safety Concept
* **OLED Frame Buffer Refresh**: SSD1306 screens use standard memory buffers. Because data is stored in memory, calling graphics routines like `drawLine()` modifies the buffer instantly but does not update the screen. Calling `display.display()` initiates high-speed I2C burst transfers to write memory modifications onto the screen registers.

## Try This! (Challenges)
1. **Dynamic Grid**: Draw horizontal dotted lines at Y coordinates corresponding to 20 °C and 30 °C as visual references.
2. **Double Speed Graphing**: Reduce the measurement delay to 1 second and the step size to 1 pixel for higher-resolution charts.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Graph is flat or doesn't update | Sensor read fails | Check DHT22 DATA connection and pull-up resistor. |
| Display is completely blank | Missing `display()` or wrong address | Ensure `display.display()` is called to push changes; verify I2C address is `0x3C`. |
| Graph does not wrap | Coordinate limits wrong | Check if `currentX >= 128` logic is properly clearing the display. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [60 - OLED SSD1306 Display Setup (I2C)](60-oled-ssd1306-display-setup.md)
- [71 - DHT22 Temp & Humidity Serial Logs](71-dht22-temp-humidity-serial.md)
- [73 - DHT22 Temperature LCD HUD](73-dht22-temperature-lcd-hud.md)
