# 140 - Multi-Sensor Air Quality HUD

Integrate an MQ-2 gas sensor and a DHT22 temperature/humidity sensor to display live ambient safety and environmental parameters on an SSD1306 I2C OLED display using the VEGA ARIES v3 board.

## Goal
Learn how to read analog inputs (MQ-2) alongside digital data streams (DHT22), render graphics and custom formatted text on a high-density OLED screen using the Adafruit SSD1306 library, and manage display updates in interpreted C++.

## What You Will Build
An indoor air quality and comfort monitor. The ARIES board measures analog air pollution index (gas, smoke, LPG) via the MQ-2 sensor on GP28 (ADC2) and temperature/humidity via the DHT22 on GPIO 12. These parameters are rendered on a 128x64 pixel OLED screen. The HUD draws a clean layout displaying Temperature, Humidity, and a Gas intensity progress indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| SSD1306 OLED Display | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Red | Sensor heater power (5V) |
| MQ-2 Sensor | GND | GND | Black | Ground reference |
| MQ-2 Sensor | AO (Analog Out) | GP28 | Purple | Analog signal to ADC2 |
| DHT22 Sensor | VCC | 3V3 | Orange | Digital sensor power (3.3V) |
| DHT22 Sensor | SDA (Data) | GPIO 12 | Purple | Digital Single-Bus Data |
| DHT22 Sensor | GND | GND | Grey | Ground reference |
| SSD1306 OLED | VCC | 3V3 | Red | Display power (3.3V) |
| SSD1306 OLED | GND | GND | Black | Ground reference |
| SSD1306 OLED | SDA | SDA0 (GP17) | Blue | I2C Data line |
| SSD1306 OLED | SCL | SCL0 (GP16) | Yellow | I2C Clock line |

> **Wiring tip:** The MQ-2 sensor requires 5V for its heating element. If powered from 3.3V, the sensor will fail to heat up and will not respond to gas concentration changes.

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_ADDR 0x3C

#define DHTPIN 12
#define DHTTYPE DHT22

DHT dht(DHTPIN, DHTTYPE);
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

const int mq2Pin = GP28; // ADC2 pin
unsigned long lastUpdate = 0;

void setup() {
  Serial.begin(115200);
  delay(1000);

  // Initialize I2C, DHT, and OLED
  Wire.begin();
  dht.begin();
  pinMode(mq2Pin, INPUT);

  if (!display.begin(SSD1306_SWITCHCAPVCC, OLED_ADDR)) {
    Serial.println("SSD1306 allocation failed");
  } else {
    Serial.println("OLED Initialized.");
  }

  display.clearDisplay();
  display.setTextSize(1);
  display.setTextColor(SSD1306_WHITE);
  display.setCursor(20, 20);
  display.print("Air Quality HUD");
  display.display();
  delay(1500);
}

void loop() {
  unsigned long currentTime = millis();

  // Update sensor readings and display every 1000 ms (1 second)
  if (currentTime - lastUpdate >= 1000) {
    int gasRaw = analogRead(mq2Pin);
    float temp = dht.readTemperature();
    float hum = dht.readHumidity();

    // Map MQ-2 raw ADC (0-1023) to a simple percentage (0-100)
    int gasPercent = (gasRaw * 100) / 1023;

    // Clear buffer
    display.clearDisplay();

    // Header Title
    display.setTextSize(1);
    display.setCursor(0, 0);
    display.print("AIR MONITOR STATUS");
    
    // Draw horizontal separator line
    display.drawLine(0, 10, 127, 10, SSD1306_WHITE);

    // Render Temp
    display.setCursor(0, 18);
    display.print("Temp: ");
    display.print(temp, 1);
    display.print(" C");

    // Render Humidity
    display.setCursor(0, 30);
    display.print("Humidity: ");
    display.print(hum, 1);
    display.print(" %");

    // Render Gas level
    display.setCursor(0, 42);
    display.print("Gas: ");
    display.print(gasRaw);
    display.print(" (");
    display.print(gasPercent);
    display.print("%)");

    // Draw visual safety bar at bottom (128 pixels wide maximum)
    int barWidth = (gasPercent * 128) / 100;
    display.fillRect(0, 56, barWidth, 6, SSD1306_WHITE);

    // Push local frame buffer to OLED
    display.display();

    // Output to Serial for debugging
    Serial.print("Temp: "); Serial.print(temp);
    Serial.print(" | Hum: "); Serial.print(hum);
    Serial.print(" | Gas Raw: "); Serial.print(gasRaw);
    Serial.print(" ("); Serial.print(gasPercent);
    Serial.println("%)");

    lastUpdate = currentTime;
  }

  delay(10);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MQ-2 Gas Sensor**, **DHT22 Sensor**, and **SSD1306 OLED Display** onto the canvas.
2. Connect the MQ-2: **VCC** to **5V**, **GND** to **GND**, and **AO** to **GP28**.
3. Connect the DHT22: **VCC** to **3V3**, **GND** to **GND**, and **SDA** to **GPIO 12**.
4. Connect the OLED: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
5. Paste the code into the editor.
6. Select **Interpreted Mode** in the simulation dropdown.
7. Click **Run** to execute the program.
8. Adjust the MQ-2 gas concentration slider on the canvas and observe the bottom progress bar change.

## Expected Output
Serial Monitor:
```
System Initialized.
OLED Initialized.
Temp: 24.8 | Hum: 50.2 | Gas Raw: 120 (11%)
Temp: 24.8 | Hum: 50.1 | Gas Raw: 450 (43%)
```

## Expected Canvas Behavior
* The OLED display prints "AIR MONITOR STATUS" on the top line.
* The screen displays temperature, humidity, and the raw/percent gas value.
* A solid bar at the bottom expands or contracts as the gas levels adjust.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `display.begin(...)` | Performs default SPI/I2C handshakes to configure internal screen charge pumps. |
| `display.clearDisplay()` | Resets the 1KB memory frame buffer inside the microcontroller. |
| `display.drawLine(...)` | Draws a single pixel line from coordinates (0,10) to (127,10). |
| `display.fillRect(...)` | Fills a solid rectangle on the bottom edge representing the gas concentration visually. |
| `display.display()` | Serializes the entire screen buffer to the OLED module over I2C. |

## Hardware & Safety Concept
* **OLED Frame Buffering**: Renders on OLEDs rely on drawing to a local buffer before updating the screen. This allows multi-element graphic updates without the screen flickering.
* **MQ-2 Warm-up Period**: In physical environments, the MQ-2 sensor's internal tin dioxide (SnO2) element takes 24-48 hours of run-time to stabilize, and a few minutes of heating upon startup before it can give reliable measurements.

## Try This! (Challenges)
1. **Critical Warning State**: If the Gas Raw reading exceeds `500`, clear the OLED screen and print a blinking large `"GAS WARNING!"` screen.
2. **Temperature Alert**: Invert the display pixels using `display.invertDisplay(true)` when temperature exceeds 35°C to flag extreme heat.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED shows static noise or is black | SSD1306 missing power | Verify that SSD1306 VCC goes to 3V3, and check the target I2C address (default `0x3C`). |
| Gas readings always high | Sensor heating phase | Wait for the virtual sensor to settle or adjust the slider to a lower point. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [139 - Multi-Sensor Environmental HUD](139-multi-sensor-environmental-hud.md) (Previous project)
- [141 - Sounding Sentry Guard](141-sounding-sentry-guard.md) (Next project)
