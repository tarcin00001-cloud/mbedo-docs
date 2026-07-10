# 119 - Pico BMP180 OLED

Build a digital altimeter dashboard that displays temperature, pressure, and relative altitude on an OLED screen.

## Goal
Learn how to interface I2C pressure sensors (BMP180) and display multiple coordinate calculations on high-resolution graphic screens.

## What You Will Build
An aviator's instrument dashboard:
- **BMP180 Sensor (GP4 SDA, GP5 SCL)**: Monitors atmospheric pressure.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature, pressure, and calculated altitude with custom layout frames.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| BMP180 Barometric Sensor | `bmp180` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | GP4 | Shared I2C Data |
| BMP180 Sensor | SCL | GP5 | Shared I2C Clock |
| BMP180 Sensor | GND | GND | Ground return |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | Shared I2C Data |
| SSD1306 OLED | SCL | GP5 | Shared I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

Adafruit_BMP085 bmp;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }

  if (!bmp.begin()) {
    display.clearDisplay();
    display.setTextSize(1);
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(10, 20);
    display.print("BMP180 Error!");
    display.display();
    while (1); // Halt if sensor missing
  }
}

void loop() {
  float temp = bmp.readTemperature();
  long pressure = bmp.readPressure();
  float altitude = bmp.readAltitude();

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("ALTIMETER HUD");

  // Display values
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  
  // Row 1: Temperature
  display.setCursor(10, 20);
  display.print("Temp    : ");
  display.print(temp, 1);
  display.print(" C");

  // Row 2: Pressure
  display.setCursor(10, 32);
  display.print("Pressure: ");
  display.print(pressure / 100);
  display.print(" hPa");

  // Row 3: Altitude
  display.setCursor(10, 48);
  display.print("Altitude: ");
  display.print(altitude, 1);
  display.print(" m");

  // Outer border frame
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(1000); // Update once per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **BMP180 Sensor**, and **SSD1306 OLED** onto the canvas.
2. Connect both BMP180 and OLED to GP4 (SDA) and GP5 (SCL) in parallel.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the sensor pressure slider on canvas and watch the temperature, pressure, and altitude update.

## Expected Output

Terminal:
```
Simulation active. Altimeter instrument panel online.
```

## Expected Canvas Behavior
* Header: White banner reading `ALTIMETER HUD` in black text.
* Row 1: `Temp    : 24.5 C`
* Row 2: `Pressure: 1013 hPa`
* Row 3: `Altitude: 112.5 m`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `pressure / 100` | Converts raw pressure values from Pascals (Pa) to hectopascals (hPa) or millibars (1 hPa = 100 Pa) for standard meteorological readings. |

## Hardware & Safety Concept: I2C Pull-up Resistors
The I2C bus uses open-drain lines, meaning devices can only pull the SDA/SCL lines `LOW`. To return the lines to a `HIGH` logic level, pull-up resistors are required. Standard breakout boards for the BMP180 and OLED include built-in 10k or 4.7k pull-up resistors. However, if you connect too many devices to the bus, the parallel resistance drops, which can distort signals. Keep total bus resistance between 2k and 5k.

## Try This! (Challenges)
1. **Target Altitude Warning**: Sound a buzzer on GP14 if the altitude changes by more than 10 meters from startup.
2. **Pressure Trend Arrow**: Display an arrow on the OLED screen pointing UP if pressure is rising, or DOWN if falling.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen stays blank on boot | I2C address conflict | Check that the BMP180 is powered and shares the same I2C lines as the OLED. Standard address is 0x77 for BMP180 and 0x3C for OLED. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [78 - Pico BMP180 Serial](../intermediate/78-pico-bmp180-serial.md)
- [79 - Pico BMP180 LCD](../intermediate/79-pico-bmp180-lcd.md)
- [116 - Pico DHT OLED HUD](116-pico-dht-oled-hud.md)
