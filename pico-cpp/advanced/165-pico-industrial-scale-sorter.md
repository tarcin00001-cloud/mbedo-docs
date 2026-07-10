# 165 - Pico Industrial Scale Sorter

Build an automated sorting conveyor that weighs packages on a load cell and routes them using separate sorting gates based on weight classifications.

## Goal
Learn how to interface HX711 load cell amplifiers, display sorting metrics and categories on OLED screens, and actuate dual-relay sorting gates.

## What You Will Build
A multi-bin industrial weight sorter:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures package weight.
- **Relay 1 (GP10)**: Reject gate for underweight packages (< 100g).
- **Relay 2 (GP11)**: Reject gate for overweight packages (> 200g).
- **SSD1306 OLED (GP4, GP5)**: Displays live weight and active bin routing (e.g. "UNDERWEIGHT", "STANDARD", or "OVERWEIGHT").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT / PD_SCK | GP14 / GP15 | Sensor lines |
| Relay 1 | IN | GP10 | Underweight reject gate |
| Relay 2 | IN | GP11 | Overweight reject gate |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>
#include "HX711.h"

const int DOUT_PIN  = 14;
const int SCK_PIN   = 15;
const int RELAY1_PIN = 10;
const int RELAY2_PIN = 11;

HX711 scale;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

float calibration_factor = 420.0;

// Sorter target boundaries
const float UNDER_LIMIT = 100.0; // Underweight limit (grams)
const float OVER_LIMIT  = 200.0; // Overweight limit (grams)

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  scale.begin(DOUT_PIN, SCK_PIN);
  scale.set_scale(calibration_factor);
  scale.tare(); // Zero the scale on startup

  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);

  digitalWrite(RELAY1_PIN, LOW);
  digitalWrite(RELAY2_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  // Read weight (average over 5 readings)
  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("SORTING CONVEYOR");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 20);
  display.print("Weight: ");
  display.print(weight, 1);
  display.print(" g");

  display.setCursor(10, 34);

  if (weight > 10.0) { // Check only if an item is on the scale
    if (weight < UNDER_LIMIT) {
      // Underweight sorting action
      digitalWrite(RELAY1_PIN, HIGH);
      digitalWrite(RELAY2_PIN, LOW);
      display.print("CLASS : UNDERWEIGHT");
      display.setCursor(10, 48);
      display.print("Gate 1: ACTUATING");
    } 
    else if (weight > OVER_LIMIT) {
      // Overweight sorting action
      digitalWrite(RELAY1_PIN, LOW);
      digitalWrite(RELAY2_PIN, HIGH);
      display.print("CLASS : OVERWEIGHT");
      display.setCursor(10, 48);
      display.print("Gate 2: ACTUATING");
    } 
    else {
      // Standard weight - pass through
      digitalWrite(RELAY1_PIN, LOW);
      digitalWrite(RELAY2_PIN, LOW);
      display.print("CLASS : STANDARD");
      display.setCursor(10, 48);
      display.print("Gates : PASS");
    }
  } else {
    // Empty
    digitalWrite(RELAY1_PIN, LOW);
    digitalWrite(RELAY2_PIN, LOW);
    display.print("CLASS : EMPTY");
  }

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **two Relays**, and **SSD1306 OLED** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relays to **GP10/GP11**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the weight potentiometer to simulate package weights and check if the correct reject gate activates.

## Expected Output

Terminal:
```
Simulation active. Industrial weighing sorter online.
```

## Expected Canvas Behavior
* Empty Scale (< 10g): OLED reads `CLASS: EMPTY`. Both relays are OFF.
* Standard Package (150g): OLED reads `CLASS: STANDARD` / `Gates: PASS`. Both relays are OFF.
* Underweight Package (50g): OLED reads `CLASS: UNDERWEIGHT` / `Gate 1: ACTUATING`. Relay 1 turns ON.
* Overweight Package (250g): OLED reads `CLASS: OVERWEIGHT` / `Gate 2: ACTUATING`. Relay 2 turns ON.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `weight < UNDER_LIMIT` | Evaluates if the package is underweight to route it to reject bin 1. |

## Hardware & Safety Concept: Industrial Sorting Gates
Industrial sorting systems route packages to different bins (such as sorting by size, weight, or ZIP code). Pneumatic reject arms or diverter gates open quickly to guide items into their correct channels. Emergency limit monitoring is critical: if a gate jams, safety sensors shut down the conveyor line to prevent package blockages.

## Try This! (Challenges)
1. **Auditory Alert**: Connect a buzzer on GP12 and sound a beep when a package is sorted to either reject bin.
2. **Interactive Calibration**: Add a push button on GP16 that zeroes the scale when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Both gates activate at the same time | Code logic error | Ensure the check conditions use `if / else if` structures so only one relay can activate at a time. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [109 - Pico Load Cell LCD](../intermediate/109-pico-loadcell-lcd.md)
- [135 - Pico Industrial Scale](135-pico-industrial-scale.md)
- [154 - Pico Industrial Scale OLED](154-pico-industrial-scale-oled.md)
