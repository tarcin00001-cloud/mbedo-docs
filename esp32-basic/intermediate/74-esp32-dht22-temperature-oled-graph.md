# 74 - ESP32 DHT22 Temperature OLED Graph

Plot a scrolling real-time historical temperature graph on an SSD1306 OLED display using readings from a DHT22 sensor.

## Goal
Learn how to build a basic graphical data buffer (history queue) in RAM, map floating-point values to screen pixel coordinates, and render pixel-level lines on an OLED.

## What You Will Build
A DHT22 sensor measures temperature on GPIO 4. The ESP32 maintains a buffer of the last 128 readings. Every 2 seconds, it draws a 128-pixel wide scrolling line graph on the SSD1306 OLED display alongside a numeric readout of the current temperature.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| 0.96″ SSD1306 OLED (I2C) | `oled` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DHT22 Sensor | DATA | GPIO4 | Yellow | Temp input |
| DHT22 Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| SSD1306 OLED | VCC | 3V3 | Red | OLED power |
| SSD1306 OLED | GND | GND | Black | Ground |
| SSD1306 OLED | SDA | GPIO21 | Blue | I2C Data |
| SSD1306 OLED | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** Share the I2C bus pins (GPIO 21 and 22) for the OLED. Make sure both the OLED and DHT22 are powered from the 3.3V pin on the ESP32 to prevent logic mismatch.

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

#define DHTPIN 4
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

// Circular history buffer for graph values (1 pixel = 1 sample)
float tempHistory[128];
int historyIndex = 0;
bool bufferFull = false;

// Graph bounds (y-axis limits in Celsius)
const float TEMP_MIN = 15.0;
const float TEMP_MAX = 45.0;

void setup() {
  Serial.begin(115200);
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED allocation failed!");
    while(true);
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Graph station booting...");
  display.display();
  
  dht.begin();
  delay(1500);
  
  // Initialize history buffer with zeros
  for (int i = 0; i < 128; i++) {
    tempHistory[i] = 0.0;
  }
}

void loop() {
  delay(2000); // Wait 2s between DHT samples
  
  float temp = dht.readTemperature();
  if (isnan(temp)) {
    Serial.println("DHT22 Error");
    return;
  }
  
  // Store reading in buffer
  tempHistory[historyIndex] = temp;
  historyIndex = (historyIndex + 1) % 128;
  if (historyIndex == 0) bufferFull = true;
  
  // Redraw screen
  display.clearDisplay();
  
  // 1. Draw numeric HUD header
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(0, 0);
  display.print("Temp: ");
  display.print(temp, 1);
  display.print(" C");
  
  // 2. Draw graph area borders (y = 16 to 63)
  display.drawRect(0, 16, 128, 48, SSD1306_WHITE);
  
  // 3. Plot temperature history lines
  // Map range [TEMP_MIN, TEMP_MAX] to screen coordinates [62, 17] (inverted y-axis)
  int startPoint = bufferFull ? historyIndex : 0;
  int numPoints = bufferFull ? 128 : historyIndex;
  
  int prevX = -1;
  int prevY = -1;
  
  for (int i = 0; i < numPoints; i++) {
    int bufferPos = (startPoint + i) % 128;
    float val = tempHistory[bufferPos];
    
    // Scale temp to y-axis coordinates
    float fraction = (val - TEMP_MIN) / (TEMP_MAX - TEMP_MIN);
    fraction = constrain(fraction, 0.0, 1.0);
    
    int x = i;
    int y = 62 - (int)(fraction * 45.0); // 45 pixels height inside border
    
    if (prevX != -1) {
      display.drawLine(prevX, prevY, x, y, SSD1306_WHITE);
    }
    
    prevX = x;
    prevY = y;
  }
  
  display.display();
  Serial.print("Plotting temp: "); Serial.println(temp);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DHT22**, and **SSD1306 OLED** onto the canvas.
2. Connect DHT22 DATA to **GPIO4**, and OLED SDA/SCL to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Slide the temperature on the DHT22 widget and watch the line plot move in response.

## Expected Output
Serial Monitor:
```
Plotting temp: 24.50
Plotting temp: 26.80
Plotting temp: 28.00
```

OLED Display layout:
```
Temp: 28.0 C
┌────────────────┐
│      /\        │  ← Scrolling historical temperature line graph
│  ___/  \       │
└────────────────┘
```

## Expected Canvas Behavior
* At boot, the screen displays "Graph station booting...".
* Every 2 seconds, a new data point is plotted from left to right.
* Once the data fills the screen width (128 samples, ~4.2 minutes), the graph scrolls smoothly to the left, continuously displaying the latest history.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `tempHistory[historyIndex] = temp` | Inserts the new sensor reading into the circular history array. |
| `62 - (int)(fraction * 45.0)` | Translates Celsius value to inverted pixel y-coordinate. |
| `display.drawLine(prevX, prevY, x, y, ...)` | Draws a line segment connecting the previous coordinate to the new one. |
| `display.drawRect(0, 16, 128, 48, ...)` | Draws the graph box borders to outline the plotting viewport. |

## Hardware & Safety Concept: Circular Buffer Storage
Logging temporal data efficiently requires a **circular buffer**. Instead of shifting every item in a large array (expensive CPU execution), we store a single write index pointer that wraps around (`historyIndex = (historyIndex + 1) % 128`). This saves execution cycles and prevents memory fragmentation in RAM.

## Try This! (Challenges)
1. **Dynamic Scaling**: Automatically adjust `TEMP_MIN` and `TEMP_MAX` based on the lowest and highest values currently inside the history buffer.
2. **Speed Select Potentiometer**: Read a potentiometer (GPIO 34) to speed up or slow down the graph sampling rate.
3. **Grid Lines**: Draw horizontal dotted grid lines representing 20 °C, 30 °C, and 40 °C across the plotting viewport.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Line graph is flat on the bottom/top | Values exceed the graph limits | Adjust `TEMP_MIN` (e.g. 10) and `TEMP_MAX` (e.g. 50) to bracket actual room/simulated temperatures |
| Display is completely black | Allocation or power issue | Verify `display.begin()` returns true; check SDA/SCL wiring |
| Graph flickers when updating | Full screen clear occurring inside loops | Make sure code updates screen at the same frequency as sensor queries (2s) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [71 - ESP32 DHT22 Temp & Humidity Serial Logs](71-esp32-dht22-temp-humidity-serial-logs.md)
- [73 - ESP32 DHT22 Temperature LCD HUD](73-esp32-dht22-temperature-lcd-hud.md)
- [60 - ESP32 OLED SSD1306 Display Setup](60-esp32-oled-ssd1306-display-setup.md)
