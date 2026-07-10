# 121 - Pico Fire System

Build an advanced fire safety system that displays alerts on an OLED, activates a warning buzzer, and switches a ventilation relay on flame detection.

## Goal
Learn how to coordinate multiple output indicators (OLED, active buzzer, and relay) in response to analog sensor threshold triggers.

## What You Will Build
An integrated industrial fire alarm node:
- **Flame Sensor (GP26)**: Monitors ambient infrared flame levels.
- **SSD1306 OLED (GP4, GP5)**: Displays normal monitoring messages or flashing fire emergency warnings.
- **Active Buzzer (GP14)**: Sounds a pulsing warning siren when fire is detected.
- **Relay Module (GP10)**: Actuates a 5V exhaust fan during alarms.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor | `phototransistor` | Yes (represented by phototransistor) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 3.3V | Power supply |
| Flame Sensor | AO (Analog) | GP26 | Flame analog level |
| Flame Sensor | GND | GND | Ground return |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | Shared I2C Data |
| SSD1306 OLED | SCL | GP5 | Shared I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |
| Active Buzzer | VCC (+) | GP14 | Sounder control |
| Active Buzzer | GND (-) | GND | Ground return |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Exhaust fan control |
| Relay Module | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int FLAME_PIN  = 26;
const int BUZZER_PIN = 14;
const int RELAY_PIN  = 10;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Threshold: Flame sensor value drops below limit near fire
const int FLAME_LIMIT = 1500;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(FLAME_PIN, INPUT);
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
  int flameVal = analogRead(FLAME_PIN);

  display.clearDisplay();

  if (flameVal < FLAME_LIMIT) {
    // FIRE ALARM ACTIVE
    digitalWrite(BUZZER_PIN, HIGH);
    digitalWrite(RELAY_PIN, HIGH); // Turn exhaust fan ON

    // Flashing OLED Alert screen (Inverted Colors)
    display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(2);
    display.setCursor(15, 10);
    display.print("!!! FIRE !!!");
    
    display.setTextSize(1);
    display.setCursor(20, 40);
    display.print("Ventilation: ON");
    
    display.display();
    delay(200);

    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(2);
    display.setCursor(15, 10);
    display.print("!!! FIRE !!!");
    
    display.setTextSize(1);
    display.setCursor(20, 40);
    display.print("Ventilation: ON");
    
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
    
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  } else {
    // SYSTEM SAFE
    digitalWrite(BUZZER_PIN, LOW);
    digitalWrite(RELAY_PIN, LOW); // Exhaust fan OFF

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(15, 5);
    display.print("SAFETY STATION");

    display.setCursor(10, 24);
    display.print("Flame Index: ");
    display.print(flameVal);

    display.setCursor(10, 40);
    display.print("Status: SECURE");

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();

    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Flame Sensor** (phototransistor), **SSD1306 OLED**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect Flame Sensor to **GP26**, OLED to **GP4/GP5**, Buzzer to **GP14**, and Relay to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the flame sensor slider on canvas to simulate a fire, and watch the OLED warning flash, the buzzer beep, and the relay activate.

## Expected Output

Terminal:
```
Simulation active. Industrial fire safety node online.
```

## Expected Canvas Behavior
| Flame sensor reading | OLED Display | GP14 (Buzzer) | GP10 (Relay) |
| --- | --- | --- | --- |
| Safe (> 2000) | `Status: SECURE` | LOW | LOW |
| Fire (< 1500) | Flashing `!!! FIRE !!!` | Pulsing HIGH/LOW | HIGH (Ventilation ON) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the relay contact to start the emergency ventilation fan to clear smoke. |

## Hardware & Safety Concept: Industrial Fire Alarm Standards
Industrial fire alarm systems must be highly reliable. Standard code requirements specify that fire safety controllers run on isolated backup battery lines so they continue working during power cuts. Actuators like sirens and ventilation fans are often controlled by solid-state relays rather than mechanical relays to prevent contacts from sparking in explosive gas areas.

## Try This! (Challenges)
1. **Latching Alarm Reset**: Add a push button on GP15. When a fire is detected, latch the alarm ON permanently until the reset button is pressed.
2. **CO2 Warning**: Add an MQ-2 Gas sensor on GP27 and trigger the alarm if either smoke OR flame is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Fire warning triggers under normal ceiling lights | Ambient light interference | Adjust the sensor orientation or lower the threshold `FLAME_LIMIT` below 1000. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [38 - Pico Flame Sensor](../../beginner/38-pico-flame-sensor.md)
- [107 - Pico Fire Station](../intermediate/107-pico-fire-station.md)
