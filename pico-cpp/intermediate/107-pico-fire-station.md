# 107 - Pico Fire Station

Build a fire warning station that displays alerts on an OLED and sounds a buzzer when a flame is detected.

## Goal
Learn how to interface analog flame sensors, display graphic alert screens on SSD1306 OLEDs, and drive active buzzers.

## What You Will Build
A fire safety monitor:
- **Flame Sensor (GP26)**: Monitors ambient infrared levels from fire.
- **Active Buzzer (GP14)**: Sounds a loud pulsing alarm when a flame is detected.
- **SSD1306 OLED (GP4, GP5)**: Displays a flashing "FIRE ALERT!" warning screen during alarms, and "System Safe" when clear.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Flame Sensor | `phototransistor` | Yes (represented by phototransistor) | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 3.3V | Power supply |
| Flame Sensor | AO (Analog) | GP26 | Analog light intensity input |
| Flame Sensor | GND | GND | Ground return |
| Active Buzzer | VCC (+) | GP14 | Alarm pin |
| Active Buzzer | GND (-) | GND | Ground return |
| SSD1306 OLED | VCC | 3.3V | Display power |
| SSD1306 OLED | SDA | GP4 | I2C Data |
| SSD1306 OLED | SCL | GP5 | I2C Clock |
| SSD1306 OLED | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int FLAME_PIN  = 26;
const int BUZZER_PIN = 14;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Threshold: Flame sensors output low voltage when light/flame is present
// Under ambient light, output is high (~4000). Near flame, drops below 1500.
const int FLAME_THRESHOLD = 1500;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(FLAME_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int flameVal = analogRead(FLAME_PIN);

  display.clearDisplay();

  if (flameVal < FLAME_THRESHOLD) {
    // Fire Alert Mode
    digitalWrite(BUZZER_PIN, HIGH);

    // Draw solid warning background
    display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(2);
    display.setCursor(15, 20);
    display.print("!!! FIRE !!!");

    display.display();
    delay(200);

    // Alternate warning screen frame to create flashing effect
    display.clearDisplay();
    display.setTextColor(SSD1306_WHITE);
    display.setCursor(15, 20);
    display.print("!!! FIRE !!!");
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
    
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
  } else {
    // System Safe Mode
    digitalWrite(BUZZER_PIN, LOW);

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    display.setCursor(20, 15);
    display.print("Fire Monitor");
    
    display.setCursor(20, 35);
    display.print("Status: Safe");
    
    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
    display.display();
    
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Flame Sensor** (represented by phototransistor), **Active Buzzer**, and **SSD1306 OLED** onto the canvas.
2. Connect Flame Sensor to **GP26**, Buzzer to **GP14**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the light level on the phototransistor to maximum brightness (simulating a flame) to trigger the warning station.

## Expected Output

Terminal:
```
Simulation active. Flame warning station online.
```

## Expected Canvas Behavior
| Flame Sensor Slider | GP14 (Buzzer) | OLED Screen Visual | Safety Status |
| --- | --- | --- | --- |
| Low Light (Safe) | LOW | Normal text frame reading `Status: Safe` | Normal |
| High Light (Flame) | Pulsing | Flashing black-and-white `!!! FIRE !!!` | **EMERGENCY WARNING** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `display.fillRect(0, 0, 128, 64, SSD1306_WHITE)` | Fills the entire OLED screen with white pixels to invert the display colors during alarms. |

## Hardware & Safety Concept: Optical Flame Detection
Flame sensors use an infrared-sensitive phototransistor to detect light wavelengths emitted by fire (typically in the 760 nm to 1100 nm range). Since sunlight and incandescent light bulbs also emit infrared light, flame sensors can trigger false alarms if exposed to direct light. Real systems combine optical infrared sensors with thermal and smoke detectors to confirm fire conditions.

## Try This! (Challenges)
1. **Ventilation Trigger**: Connect a relay on GP10 and activate it when a flame is detected to run exhaust fans.
2. **Alert Logger**: Log "WARNING: FIRE DETECTED!" messages to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alert triggers constantly under normal light | Threshold set too high | Check the ambient infrared light levels and lower `FLAME_THRESHOLD` below 1000. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [38 - Pico Flame Sensor](../../beginner/38-pico-flame-sensor.md)
- [60 - Pico OLED Setup](60-pico-oled-setup.md)
