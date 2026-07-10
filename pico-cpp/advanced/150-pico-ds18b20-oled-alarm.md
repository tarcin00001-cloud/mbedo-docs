# 150 - Pico DS18B20 OLED Alarm

Build a dual-probe temperature monitoring station that displays readings on an OLED and sounds a buzzer if temperatures become unbalanced.

## Goal
Learn how to read multiple digital probes (DS18B20) on a shared One-Wire bus, calculate temperature differences, and display warning logs on SSD1306 OLED displays.

## What You Will Build
A temperature imbalance safety console:
- **Sensor 1 & Sensor 2 (GP12 One-Wire)**: Share the same DQ data line.
- **SSD1306 OLED (GP4, GP5)**: Displays both temperatures and imbalance metrics.
- **Active Buzzer (GP14)**: Sounds a warning if the temperature difference exceeds 5.0°C.
- **Relay Module (GP10)**: Toggles a mixing pump or heater cut-off safety relay during alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temp Sensors | `ds18b20` | Yes (two sensors) | Yes (two waterproof probes) |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 #1 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| DS18B20 #2 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| Relay Module | IN | GP10 | Safety shutdown line |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int ONE_WIRE_BUS = 12;
const int BUZZER_PIN   = 14;
const int RELAY_PIN    = 10;

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
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);

  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);

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
  display.setCursor(18, 3);
  display.print("TEMP BALANCE HUD");

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

  // Calculate delta difference
  display.setCursor(10, 48);
  if (t1 != DEVICE_DISCONNECTED_C && t2 != DEVICE_DISCONNECTED_C) {
    float diff = t1 - t2;
    if (diff < 0) { diff = -diff; }

    if (diff > 5.0) {
      // Critical Imbalance
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(RELAY_PIN, HIGH); // Turn mixing pump/heater OFF
      display.print("DIFF: !!! UNBALANCED");
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(RELAY_PIN, LOW);
      display.print("DIFF: BALANCED");
    }
  } else {
    display.print("DIFF: ERROR");
  }

  // Outer border frame
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(1500); // Update once per 1.5 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two DS18B20 Sensors**, **SSD1306 OLED**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect both DS18B20 sensors' data pins to **GP12** in parallel (add a 4.7k resistor to 3V3). Connect OLED to **GP4/GP5**, Buzzer to **GP14**, and Relay to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the temperature sliders on both sensors and watch the OLED dashboard and alert systems update.

## Expected Output

Terminal:
```
Simulation active. Dual probe temperature balancing safety system online.
```

## Expected Canvas Behavior
* Both Equal (24°C): OLED reads `DIFF: BALANCED`. Relay and Buzzer are OFF.
* Imbalance (Probe 1 24°C, Probe 2 30°C): OLED reads `DIFF: !!! UNBALANCED`. Relay turns ON, buzzer beeps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `diff > 5.0` | Evaluates if the temperature difference between the two probes exceeds the safety threshold. |

## Hardware & Safety Concept: Industrial Thermal Protection
In chemical processing, food manufacturing, and HVAC systems, heating loads must be distributed evenly. If two monitoring probes in the same liquid tank show a difference of more than 5°C, it indicates poor mixing, a heating element failure, or a pipe blockage. Safety controllers monitor this delta and shut down heaters to prevent product damage or boiling.

## Try This! (Challenges)
1. **Auditory Indicator**: Sound different chime patterns based on how large the temperature difference is.
2. **Reverse Refill Control**: Connect a relay on GP10 and turn it ON if either probe drops below 15°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display shows "Probe 2: ERROR" | Data connection loose | Check that the data pins of both sensors are tied to the exact same physical node on GP12. |

## Mode Notes
This multi-device I2C and One-Wire project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [81 - Pico DS18B20 Serial](../intermediate/81-pico-ds18b20-serial.md)
- [83 - Pico DS18B20 LCD](../intermediate/83-pico-ds18b20-lcd.md)
- [120 - Pico DS18B20 OLED](../intermediate/120-pico-ds18b20-oled.md)
