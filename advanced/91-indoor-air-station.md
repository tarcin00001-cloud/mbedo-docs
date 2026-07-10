# 91 - Indoor Air Station

Build a comprehensive indoor air quality dashboard that reads temperature, humidity (DHT22), barometric pressure (BMP180), and gas/smoke (MQ-2), displaying all four readings simultaneously on a 128x64 OLED screen.

## Goal
Learn how to render a formatted multi-parameter dashboard on an SSD1306 OLED display, cycling between multiple data sources and laying out text precisely using `setCursor` coordinates.

## What You Will Build
The OLED shows a compact 4-row dashboard:
```
Temp: 24C  Hum:60%
Press: 1013 hPa
Gas:  120  (OK)
```
All readings refresh every 2 seconds. If the gas reading exceeds a threshold, the gas line shows `(ALERT)` instead of `(OK)`.

**Why these pins?** SDA/SCL (A4/A5) are the I2C bus shared by OLED and BMP180. Pin D2 is DHT22. Pin A0 is the MQ-2 analog line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2_sensor` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled_display` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Single-wire data |
| DHT22 Sensor | GND | GND | Ground reference |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Data bus |
| BMP180 Sensor | SCL | A5 | I2C Clock bus |
| BMP180 Sensor | GND | GND | Ground reference |
| MQ-2 Sensor | VCC | 5V | Power supply |
| MQ-2 Sensor | A0 | A0 | Analog output |
| MQ-2 Sensor | GND | GND | Ground reference |
| SSD1306 OLED | VCC | 3.3V | Power supply |
| SSD1306 OLED | SDA | A4 | I2C Data bus (shared) |
| SSD1306 OLED | SCL | A5 | I2C Clock bus (shared) |
| SSD1306 OLED | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET -1
#define OLED_ADDR 0x3C

const int DHT_PIN  = 2;
const int GAS_PIN  = A0;
const int DHT_TYPE = DHT22;
const int GAS_THRESHOLD = 500;

Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);
Adafruit_BMP085 bmp;
DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  Serial.begin(9600);
  dht.begin();
  bmp.begin();
  
  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("OLED init failed!");
    while (true) {} // Halt if display not found
  }
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(20, 24);
  display.println("Indoor Air Station");
  display.display();
  delay(2000);
  
  Serial.println("Indoor Air Station Active");
}

void loop() {
  float tempC    = dht.readTemperature();
  float humidity = dht.readHumidity();
  float pressure = bmp.readPressure() / 100.0;
  int   gasLevel = analogRead(GAS_PIN);
  
  Serial.print("T:"); Serial.print(tempC);
  Serial.print(" H:"); Serial.print(humidity);
  Serial.print(" P:"); Serial.print(pressure);
  Serial.print(" G:"); Serial.println(gasLevel);
  
  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  
  // Row 1: Temperature and Humidity
  display.setCursor(0, 0);
  display.print("Temp:");
  display.print(tempC, 0);
  display.print("C  Hum:");
  display.print(humidity, 0);
  display.print("%");
  
  // Row 2: Pressure
  display.setCursor(0, 16);
  display.print("Press:");
  display.print(pressure, 0);
  display.print("hPa");
  
  // Row 3: Gas level and status
  display.setCursor(0, 32);
  display.print("Gas:");
  display.print(gasLevel);
  if (gasLevel > GAS_THRESHOLD) {
    display.print(" (ALERT)");
  } else {
    display.print(" (OK)   ");
  }
  
  // Row 4: Status bar
  display.setCursor(0, 48);
  display.print("Indoor Air Monitor");
  
  display.display();
  delay(2000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **BMP180 Sensor**, **MQ-2 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect all sensors per the wiring table.
3. Paste the code into the editor.
4. Select the **Compiled Arduino Uno** runtime.
5. Click **Build and Run** (or **Run**).
6. Adjust individual sensor sliders and watch the OLED dashboard update in real-time.

## Expected Output

Terminal:
```
Indoor Air Station Active
T:24.00 H:60.00 P:1013.00 G:120
```

OLED Display:
```
Temp:24C  Hum:60%
Press:1013hPa
Gas:120 (OK)
Indoor Air Monitor
```

## Expected Canvas Behavior

| Sensor Input | OLED Row | Display Output |
| --- | --- | --- |
| DHT22 Temp: 24 C | Row 1 | `Temp:24C Hum:60%` |
| BMP180: 1013 hPa | Row 2 | `Press:1013hPa` |
| MQ-2 Gas: 120 | Row 3 | `Gas:120 (OK)` |
| MQ-2 Gas: 700 | Row 3 | `Gas:700 (ALERT)` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `display.setCursor(0, 16)` | Positions the cursor at x=0, y=16 pixels (second row, offset by 16px line height). |
| `display.clearDisplay()` | Wipes the display buffer before redrawing. Required every loop to prevent ghost text from previous values. |
| `display.display()` | Flushes the in-memory buffer to the physical OLED screen pixels. Nothing shows without this call. |

## Hardware & Safety Concept: Pixel Buffers and Double Buffering
The SSD1306 OLED library maintains a **frame buffer** in RAM (128 x 64 / 8 = 1024 bytes).
- All your `print()` and `drawLine()` calls write to this internal buffer in RAM.
- Only when you call `display.display()` is the entire buffer transferred to the display over I2C.
- This **double-buffer** technique prevents flickering, because the screen only updates once per frame, not after every individual character is drawn.

## Try This! (Challenges)
1. **Alert Inversion**: When gas level is in ALERT state, call `display.fillRect(0, 32, 128, 10, SSD1306_WHITE)` and `display.setTextColor(SSD1306_BLACK)` to draw the gas row as inverted white-on-black to make it visually prominent.
2. **Icon Bitmaps**: Create a small 8x8 temperature thermometer bitmap using `display.drawBitmap()` and render it next to the temperature reading.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED stays blank after run | OLED address mismatch | Check the I2C address; try `0x3C` or `0x3D`. |
| Text overlap / ghost values | Missing `clearDisplay()` before drawing | Ensure `display.clearDisplay()` is the first call in the render section of your loop. |

## Mode Notes
This project **requires the Compiled Arduino Uno** runtime. The `Adafruit_SSD1306` library uses dynamic RAM allocation (`malloc`) and hardware-specific bitbang routines that are not replicated in the interpreted mode.

## Related Projects
- [52 - Humidity Display OLED](../intermediate/52-humidity-display-oled.md)
- [86 - Weather Station](86-weather-station.md)
- [87 - Air Quality Monitor](87-air-quality-monitor.md)
- [92 - Rain and Wind Sim](92-rain-and-wind-sim.md)
