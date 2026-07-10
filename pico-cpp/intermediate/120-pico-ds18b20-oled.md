# 120 - Pico DS18B20 OLED

Build a dual-probe liquid temperature monitor dashboard that displays readings from two sensors on an OLED screen.

## Goal
Learn how to read multiple DS18B20 sensors on a shared One-Wire bus and align text readouts inside frames on SSD1306 OLED displays.

## What You Will Build
A dual-point liquid temperature dashboard:
- **Sensor 1 & Sensor 2 (GP12 One-Wire bus)**: Share the same data wire.
- **SSD1306 OLED (GP4, GP5)**: Displays Sensor 1 temperature (e.g. "PROBE 1: 24.5 C") and Sensor 2 temperature in parallel boxes with safety alerts.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temp Sensors | `ds18b20` | Yes (two sensors) | Yes (two waterproof probes) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 #1 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| DS18B20 #2 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| Both Sensors | VCC | 3.3V | Parallel power (with 4.7k pull-up to 3.3V) |
| Both Sensors | GND | GND | Shared ground return |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int ONE_WIRE_BUS = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  sensors.begin();
  
  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  sensors.requestTemperatures();
  
  float t1 = sensors.getTempCByIndex(0);
  float t2 = sensors.getTempCByIndex(1);

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(15, 3);
  display.print("TEMP CONTROLLER");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Probe 1 display area
  display.setCursor(10, 20);
  if (t1 != DEVICE_DISCONNECTED_C) {
    display.print("Probe 1: ");
    display.print(t1, 1);
    display.print(" C");
  } else {
    display.print("Probe 1: ERROR");
  }

  // Probe 2 display area
  display.setCursor(10, 34);
  if (t2 != DEVICE_DISCONNECTED_C) {
    display.print("Probe 2: ");
    display.print(t2, 1);
    display.print(" C");
  } else {
    display.print("Probe 2: ERROR");
  }

  // Draw warning if difference is critical (> 5C)
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    float diff = t1 - t2;
    if (diff < 0) { diff = -diff; }
    
    display.setCursor(10, 48);
    if (diff > 5.0) {
      display.print("DIFF: !!! UNBALANCED");
    } else {
      display.print("DIFF: BALANCED");
    }
  }

  // Outer border frame
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(1500); // Update once per 1.5 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two DS18B20 Sensors**, and **SSD1306 OLED** onto the canvas.
2. Connect both DS18B20 sensors' data pins to **GP12** in parallel (add a 4.7k resistor to 3V3). Connect OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature sliders on both sensors on canvas and observe the OLED dashboard update.

## Expected Output

Terminal:
```
Simulation active. Dual probe temperature dashboard online.
```

## Expected Canvas Behavior
* Header: White banner reading `TEMP CONTROLLER` in black text.
* Row 1: `Probe 1: 24.5 C`
* Row 2: `Probe 2: 26.1 C`
* Row 3: `DIFF: BALANCED` (updates to `DIFF: !!! UNBALANCED` if difference is > 5°C).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `diff > 5.0` | Computes the temperature difference between the two probes and displays a warning if they are unbalanced. |

## Hardware & Safety Concept: Industrial Temperature Balancing
In chemical processing, food manufacturing, and HVAC systems, heating loads must be distributed evenly. If two monitoring probes in the same liquid tank show a difference of more than 5°C, it indicates poor mixing, a heating element failure, or a pipe blockage. Safety controllers monitor this delta and shut down heaters to prevent product damage or boiling.

## Try This! (Challenges)
1. **Audio Alarm**: Connect a buzzer on GP14 and beep if the temperature difference exceeds 5°C.
2. **Refill Control**: Connect a relay on GP10 and turn it ON if either probe drops below 15°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "Probe 2: ERROR" | Data connection loose | Check that the data pins of both sensors are tied to the exact same physical node on GP12. |

## Mode Notes
This multi-device I2C and One-Wire project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [81 - Pico DS18B20 Serial](../intermediate/81-pico-ds18b20-serial.md)
- [83 - Pico DS18B20 LCD](../intermediate/83-pico-ds18b20-lcd.md)
- [116 - Pico DHT OLED HUD](116-pico-dht-oled-hud.md)
