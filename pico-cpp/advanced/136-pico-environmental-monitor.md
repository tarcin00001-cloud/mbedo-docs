# 136 - Pico Environmental Monitor

Build an environmental monitoring station that logs climate and gas levels and displays safety alarms on an OLED screen.

## Goal
Learn how to pool climate and gas sensor telemetry, display multi-line dashboards on OLED screens, and activate warning buzzers.

## What You Will Build
An indoor air quality station:
- **DHT22 Sensor (GP12)**: Measures ambient temperature and humidity.
- **MQ-2 Gas Sensor (GP26)**: Monitors smoke and combustible gas levels.
- **SSD1306 OLED (GP4, GP5)**: Displays temperature, humidity, gas index, and safety warnings.
- **Active Buzzer (GP14)**: Beeps when gas levels exceed the safety limit.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| MQ-2 Sensor | AO | GP26 | Gas index input |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN    = 12;
const int MQ2_PIN    = 26;
const int BUZZER_PIN = 14;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Safety gas threshold limit
const int GAS_ALARM_LIMIT = 1800;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
  pinMode(MQ2_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();
  int gas = analogRead(MQ2_PIN);

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    // Draw header bar
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(12, 3);
    display.print("AIR MONITOR STATION");

    // Display values
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);

    // Row 1: Temp & Humidity
    display.setCursor(8, 20);
    display.print("T: ");
    display.print(temp, 1);
    display.print("C | H: ");
    display.print(humid, 1);
    display.print("%");

    // Row 2: Gas Index
    display.setCursor(8, 34);
    display.print("Gas Index: ");
    display.print(gas);

    // Row 3: Safety Status Alert
    display.setCursor(8, 48);
    if (gas > GAS_ALARM_LIMIT) {
      display.print("STATUS: !!! GAS ALERT");
      
      // Sound alarm beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(100);
      digitalWrite(BUZZER_PIN, LOW);
    } else {
      display.print("STATUS: SECURE");
      digitalWrite(BUZZER_PIN, LOW);
    }

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
  }

  delay(1900); // Balance delay to match DHT22 2-second sampling interval
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **MQ-2 Sensor**, **SSD1306 OLED**, and **Active Buzzer** onto the canvas.
2. Connect DHT22 to **GP12**, MQ-2 to **GP26**, OLED to **GP4/GP5**, and Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the Gas PPM slider above 1800 to hear the alarm beep and watch the OLED warn of gas alerts.

## Expected Output

Terminal:
```
Simulation active. Environmental monitor station online.
```

## Expected Canvas Behavior
* Normal state: LCD reads `STATUS: SECURE`. Buzzer is OFF.
* Gas leak (> 1800): LCD reads `STATUS: !!! GAS ALERT`, buzzer beeps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `gas > GAS_ALARM_LIMIT` | Evaluates if the air quality index is unsafe to trigger warning screens and buzzers. |

## Hardware & Safety Concept: Multi-Gas Sensing Limitations
MEMS-based gas sensors (such as the MQ-2) are sensitive to multiple gases (including liquefied petroleum gas (LPG), propane, methane, hydrogen, and smoke). While they are cheap and easy to use, they cannot identify which specific gas is leaking. For toxic gas detection (like Carbon Monoxide), highly specific electrochemical sensors are used instead.

## Try This! (Challenges)
1. **Relay Fan Actuator**: Connect a relay on GP10 and turn ON an exhaust fan if a gas alert is triggered.
2. **Fahrenheit Toggle**: Connect a button on GP16 to switch display units between Celsius and Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen stays stuck blank | I2C address conflict | Verify that the sensor and display SDA/SCL lines are shared correctly on GP4/GP5 in parallel. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [74 - Pico DHT OLED](../intermediate/74-pico-dht-oled.md)
- [98 - Pico Gas Leak](../intermediate/98-pico-gas-leak.md)
- [117 - Pico Gas OLED](../intermediate/117-pico-gas-oled.md)
