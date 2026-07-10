# 154 - Pico Industrial Scale OLED

Build an industrial checkweigher conveyor sorting gate that displays weight measurements and PASS/REJECT status on a graphical OLED.

## Goal
Learn how to interface high-precision ADC modules (HX711), display real-time weight values and sorting status on OLED screens, and trigger sorting relays.

## What You Will Build
An automated conveyor checkweigher:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures item weight in grams.
- **Relay Module (GP10)**: Actuates a 5V reject arm to push bad items off the belt.
- **SSD1306 OLED (GP4, GP5)**: Displays the item weight, a visual bar graph, and sorting classifications.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT / PD_SCK | GP14 / GP15 | Sensor lines |
| Relay Module | IN | GP10 | Reject gate control |
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
const int RELAY_PIN = 10;

HX711 scale;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

float calibration_factor = 420.0;

// Standard weight setpoint (e.g. 150 grams target +/- 15g tolerance)
const float TARGET_WEIGHT = 150.0;
const float TOLERANCE     = 15.0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  scale.begin(DOUT_PIN, SCK_PIN);
  scale.set_scale(calibration_factor);
  scale.tare(); // Zero the scale on startup

  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Keep reject gate closed

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  // Read weight (average over 5 readings)
  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  // Map weight (0-300g) to screen bar width (0-128 pixels)
  int barWidth = weight * 128 / 300;
  if (barWidth > 128) { barWidth = 128; }

  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(18, 3);
  display.print("CHECKWEIGHER HUD");

  // Display weight
  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);
  display.setCursor(10, 20);
  display.print("Weight: ");
  display.print(weight, 1);
  display.print(" g");

  // Sorting logic
  display.setCursor(10, 32);
  if (weight > 10.0) { // Check only if an item is on the scale
    float diff = weight - TARGET_WEIGHT;
    if (diff < 0) { diff = -diff; }

    if (diff <= TOLERANCE) {
      // PASS
      digitalWrite(RELAY_PIN, LOW);
      display.print("STATUS: PASS (OK)");
    } else {
      // REJECT
      digitalWrite(RELAY_PIN, HIGH);
      display.print("STATUS: !!! REJECT !!!");
      
      // Draw warning screen and display
      display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
      display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
      display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
      display.display();

      delay(1500); // Hold reject arm
      digitalWrite(RELAY_PIN, LOW);
    }
  } else {
    // Empty
    digitalWrite(RELAY_PIN, LOW);
    display.print("STATUS: EMPTY");
  }

  // Draw bar graph border
  display.drawRect(0, 46, 128, 14, SSD1306_WHITE);
  if (barWidth > 0) {
    display.fillRect(2, 48, barWidth - 4, 10, SSD1306_WHITE);
  }
  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(400); // Update twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **Relay**, and **SSD1306 OLED** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relay to **GP10**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weight potentiometer to simulate package weights and check if the sorting relay triggers.

## Expected Output

Terminal:
```
Simulation active. Checkweigher OLED console online.
```

## Expected Canvas Behavior
* No Package (Weight < 10g): OLED reads `STATUS: EMPTY`. Relay is OFF.
* Target Package (Weight 145g): OLED reads `Weight: 145.0 g` / `STATUS: PASS (OK)`. Relay remains OFF.
* Faulty Package (Weight 100g): OLED reads `Weight: 100.0 g` / `STATUS: !!! REJECT !!!`. Relay turns ON (Reject arm active) for 1.5 seconds.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `diff <= TOLERANCE` | Checks if the measured weight is within the acceptable target range before approving passage. |

## Hardware & Safety Concept: Industrial Checkweighers
Industrial checkweighers verify package weights on high-speed conveyor lines (e.g. food packages or medicine containers) to prevent shipping underfilled boxes. If a package is underweight, a pneumatic cylinder reject arm acts immediately to push the bad package into a separate reject bin without stopping the main conveyor line.

## Try This! (Challenges)
1. **Auditory Alert**: Connect a buzzer on GP12 and play an alarm beep when a package is rejected.
2. **Interactive Tare**: Add a push button on GP16 that zeroes the scale when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reject arm triggers on empty scale | Zero calibration drift | Ensure the scale tares correctly on boot. You can also increase the empty scale detection threshold in code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [109 - Pico Load Cell LCD](../intermediate/109-pico-loadcell-lcd.md)
- [135 - Pico Industrial Scale](135-pico-industrial-scale.md)
