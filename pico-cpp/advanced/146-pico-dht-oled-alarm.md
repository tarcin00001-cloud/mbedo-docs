# 146 - Pico DHT OLED Alarm

Build an environmental monitoring station that alerts users via OLED and active buzzers when temperature thresholds are crossed.

## Goal
Learn how to parse digital climate sensors (DHT22), display warnings on OLED screens, and activate warning buzzers.

## What You Will Build
An over-temperature environmental monitor:
- **DHT22 Sensor (GP12)**: Monitors temperature and humidity.
- **SSD1306 OLED (GP4, GP5)**: Displays live temperature/humidity readouts and warning screens.
- **Active Buzzer (GP14)**: Sounds a pulsing siren when the temperature exceeds 32.0°C.
- **Relay Module (GP10)**: Actuates a 5V exhaust fan during alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | SDA | GP12 | Climate sensor (requires 10k pull-up) |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| Active Buzzer | VCC (+) | GP14 | Alarm siren |
| Relay Module | IN | GP10 | Exhaust fan control |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <DHT.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int DHT_PIN    = 12;
const int BUZZER_PIN = 14;
const int RELAY_PIN  = 10;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

const float TEMP_LIMIT = 32.0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  dht.begin();
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
  float temp = dht.readTemperature();
  float humid = dht.readHumidity();

  if (!isnan(temp) && !isnan(humid)) {
    display.clearDisplay();

    if (temp > TEMP_LIMIT) {
      // OVER-TEMPERATURE ALARM ACTIVE
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(RELAY_PIN, HIGH); // Turn exhaust fan ON

      // Flashing alert screen (Inverted Colors)
      display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
      display.setTextColor(SSD1306_BLACK);
      display.setTextSize(2);
      display.setCursor(10, 10);
      display.print("OVER-TEMP!");
      
      display.setTextSize(1);
      display.setCursor(10, 42);
      display.print("Temp: ");
      display.print(temp, 1);
      display.print(" C (Cooling ON)");
      
      display.display();
      delay(200);

      display.clearDisplay();
      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(2);
      display.setCursor(10, 10);
      display.print("OVER-TEMP!");
      
      display.setTextSize(1);
      display.setCursor(10, 42);
      display.print("Temp: ");
      display.print(temp, 1);
      display.print(" C (Cooling ON)");
      
      display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
      display.display();
      
      digitalWrite(BUZZER_PIN, LOW);
      delay(200);
    } else {
      // SYSTEM SECURE
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(RELAY_PIN, LOW); // Exhaust fan OFF

      display.setTextColor(SSD1306_WHITE);
      display.setTextSize(1);
      display.setCursor(15, 5);
      display.print("CLIMATE HUD");

      display.setCursor(10, 24);
      display.print("Temp: ");
      display.print(temp, 1);
      display.print(" C");

      display.setCursor(10, 40);
      display.print("Hum : ");
      display.print(humid, 1);
      display.print(" %");

      display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
      display.display();

      delay(1600); // Balance delay to match DHT22 rate
    }
  } else {
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22**, **SSD1306 OLED**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect DHT22 to **GP12**, OLED to **GP4/GP5**, Buzzer to **GP14**, and Relay to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature slider on the DHT22 above 32°C and observe the OLED alert screen, buzzer sound, and relay activate.

## Expected Output

Terminal:
```
Simulation active. Climate safety alert engine online.
```

## Expected Canvas Behavior
* Normal state (Temp < 32 C): OLED reads `CLIMATE HUD`, showing temperature and humidity. Relay is OFF.
* Over-temp alarm (Temp > 32 C): Screen flashes `OVER-TEMP!`, Relay turns ON (Ventilation ON), buzzer beeps.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > TEMP_LIMIT` | Checks if the measured temperature is above the safety limit to activate alarms. |

## Hardware & Safety Concept: Industrial Cooling Control Loops
Industrial HVAC systems use feedback loop logic to control temperature. If temperature thresholds are breached, the system shuts down heaters, sounds alerts, and opens exhaust dampers. To protect equipment from damage, these safety control loops run independently of other software functions.

## Try This! (Challenges)
1. **Critical Humidity Warning**: Trigger the alarm if humidity levels exceed 80%.
2. **Lockout Latch**: Latch the alarm ON permanently if a breach is detected until a reset button (GP15) is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm beeps constantly even when cool | Threshold set too low | Verify the value of `TEMP_LIMIT` in code and make sure it is set higher than room temperature. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [72 - Pico DHT Alarm](../intermediate/72-pico-dht-alarm.md)
- [116 - Pico DHT OLED HUD](../intermediate/116-pico-dht-oled-hud.md)
- [121 - Pico Fire System](121-pico-fire-system.md)
