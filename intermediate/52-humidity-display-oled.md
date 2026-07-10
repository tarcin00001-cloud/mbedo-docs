# 52 - Humidity Display OLED

Display real-time humidity levels from a DHT22 sensor on a 128x64 I2C OLED display screen.

## Goal
Learn how to use the standard `Adafruit_SSD1306` and `Adafruit_GFX` libraries to initialize and draw text/data on a high-resolution OLED screen.

## What You Will Build
The Arduino reads environmental data from the DHT22. It clears the OLED display and writes a formatted readout (e.g., "Humidity: 55.20%") in clear, graphical text that updates every 1.5 seconds.

**Why D2, A4, and A5?** Pin D2 reads the DHT22. Pins A4 (SDA) and A5 (SCL) comprise the I2C bus used to send high-resolution graphics data to the OLED screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| OLED I2C 128x64 | `oled_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (pull-up for DHT data) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| OLED Screen | VCC | 5V | Power supply (5V) |
| OLED Screen | SDA | A4 | I2C Serial Data line |
| OLED Screen | SCL | A5 | I2C Serial Clock line |
| OLED Screen | GND | GND | Ground reference |

> **Important Power Note:** Ensure `VCC` and `GND` are connected properly. An unpowered or incorrectly wired OLED display will return an I2C NACK (Negative Acknowledge) and fail to initialize, matching real hardware behavior.

## Code
```cpp
#include <DHT.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128 // OLED display width, in pixels
#define SCREEN_HEIGHT 64 // OLED display height, in pixels
#define OLED_RESET    -1 // Reset pin (unused on most I2C modules)

// Initialize SSD1306 display on I2C bus
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const int DHT_PIN = 2;
const int DHT_TYPE = DHT22;

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  Serial.begin(9600);
  dht.begin();
  
  // Start OLED on address 0x3C
  if(!display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    Serial.println("Error: SSD1306 OLED allocation failed");
    for(;;); // Halt program
  }
  
  display.clearDisplay();
  display.display();
  delay(1000);
}

void loop() {
  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();
  
  // Clear the buffer
  display.clearDisplay();
  
  if (isnan(humidity) || isnan(tempC)) {
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println("Sensor Error!");
  } else {
    // Header
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(0, 0);
    display.println("ENVIRONMENT STATUS");
    display.println("---------------------");
    
    // Draw Humidity
    display.setCursor(0, 20);
    display.println("Humidity:");
    display.setTextSize(2); // Large font
    display.print(humidity);
    display.println(" %");
    
    // Draw Temp (small)
    display.setTextSize(1);
    display.setCursor(0, 50);
    display.print("Temp: ");
    display.print(tempC);
    display.print(" C");
  }
  
  // push buffer to the physical screen
  display.display();
  
  delay(1500);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **OLED I2C** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect OLED **VCC** to Arduino **5V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
4. Paste the code into the editor.
5. Select the **Compiled Arduino Uno** runtime (OLED libraries require compiler support).
6. Click **Build and Run** (or **Run**).
7. Double-click the DHT22 sensor, adjust the sliders, and watch the values display on the OLED graphic screen on the canvas.

## Expected Output
The OLED screen lights up and displays:
```
ENVIRONMENT STATUS
---------------------
Humidity:
55.20 %
Temp: 24.50 C
```

## Expected Canvas Behavior

| Humidity Slider | Temperature Slider | OLED Screen Rendered Output |
| --- | --- | --- |
| 45.0% | 22.0 C | Large "45.00 %" with small "Temp: 22.00 C" |
| 78.5% | 30.0 C | Large "78.50 %" with small "Temp: 30.00 C" |

The OLED graphics update every 1.5 seconds.

## Code Walkthrough

| Class / Function | What It Does |
| --- | --- |
| `Adafruit_SSD1306 display(...)` | Creates the graphical driver object defining display resolution and communication interface (Wire I2C). |
| `display.begin(...)` | Sets up memory buffers and confirms communication with the screen controller chip at I2C address `0x3C`. |
| `display.clearDisplay()` | Clears the local microchip graphics buffer. |
| `display.display()` | Pushes the local graphics buffer over the I2C wires, updating the physical pixels on the screen. |

## Hardware & Safety Concept: Graphics Framebuffers
A 128x64 monochrome OLED screen contains 8,192 individual pixels. 
- To avoid slow character-by-character updates, the Arduino allocates a **framebuffer** in its RAM (1,024 bytes of memory, or 1/2 of the total RAM on the Uno).
- Drawing commands (like `print()`, `drawPixel()`, or `drawLine()`) only write data to this RAM buffer. Calling `display.display()` transmits the entire 1KB buffer over I2C in one quick transaction.

## Try This! (Challenges)
1. **Fahrenheit Conversion**: Modify the code to display Fahrenheit.
2. **Display Warning Graphic**: If the humidity exceeds 70%, write a large warning message: "MOLD RISK!" on the screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Screen remains completely black | Incorrect wiring, lack of power, or compile error | Ensure OLED VCC goes to 5V and GND goes to GND. Verify you have selected the **Compiled Arduino Uno** runtime; the code will not initialize in interpreted mode. |
| OLED initialization fails | I2C address mismatch | Verify your physical display address is `0x3C`. |

## Mode Notes
Due to the large memory footprint and templates used in Adafruit's GFX and SSD1306 libraries, this project **requires the Compiled Arduino Uno** runtime.

## Related Projects
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [51 - Temp Display LCD](51-temp-display-lcd.md)
- [53 - Dual Sensor LCD](53-dual-sensor-lcd.md)
