# 193 - Pico Industrial Scale Sorter Logger

Build an automated sorting conveyor that weighs packages on a load cell, routes them using separate relay sorting gates based on weight, updates an OLED, and logs events over Bluetooth.

## Goal
Learn how to interface HX711 load cell amplifiers, display sorting metrics and categories on OLED screens, actuate dual-relay sorting gates, and stream Bluetooth telemetry logs.

## What You Will Build
A wireless multi-bin weight sorter:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures package weight.
- **Relay 1 (GP10)**: Reject gate for underweight packages (< 100g).
- **Relay 2 (GP11)**: Reject gate for overweight packages (> 200g).
- **SSD1306 OLED (GP4, GP5)**: Displays live weight and active bin routing.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless event logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT / PD_SCK | GP14 / GP15 | Sensor lines |
| Relay 1 | IN | GP10 | Underweight reject gate |
| Relay 2 | IN | GP11 | Overweight reject gate |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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
  Serial1.begin(9600); // Bluetooth hardware UART

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

  // Print CSV Header to Bluetooth
  Serial1.println("Weight_g,Sort_code");
  delay(1000);
}

void loop() {
  // Read weight (average over 5 readings)
  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  int sortCode = 0; // 0 = Empty/Standard, 1 = Underweight, 2 = Overweight
  char classText[15] = "STANDARD";

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
      digitalWrite(RELAY1_PIN, HIGH);
      digitalWrite(RELAY2_PIN, LOW);
      sortCode = 1;
      classText[0] = 'U'; classText[1] = 'N'; classText[2] = 'D'; classText[3] = 'E'; classText[4] = 'R'; classText[5] = '\0';
      
      display.print("CLASS : UNDERWEIGHT");
      display.setCursor(10, 48);
      display.print("Gate 1: ACTUATING");
    } 
    else if (weight > OVER_LIMIT) {
      digitalWrite(RELAY1_PIN, LOW);
      digitalWrite(RELAY2_PIN, HIGH);
      sortCode = 2;
      classText[0] = 'O'; classText[1] = 'V'; classText[2] = 'E'; classText[3] = 'R'; classText[4] = '\0';

      display.print("CLASS : OVERWEIGHT");
      display.setCursor(10, 48);
      display.print("Gate 2: ACTUATING");
    } 
    else {
      digitalWrite(RELAY1_PIN, LOW);
      digitalWrite(RELAY2_PIN, LOW);
      sortCode = 0;
      display.print("CLASS : STANDARD");
      display.setCursor(10, 48);
      display.print("Gates : PASS");
    }
  } else {
    digitalWrite(RELAY1_PIN, LOW);
    digitalWrite(RELAY2_PIN, LOW);
    display.print("CLASS : EMPTY");
  }

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  // Stream CSV logs over Bluetooth
  Serial1.print(weight, 2);
  Serial1.print(",");
  Serial1.println(sortCode);

  // Hold gate active during check wait
  if (sortCode != 0) {
    delay(1500);
    digitalWrite(RELAY1_PIN, LOW);
    digitalWrite(RELAY2_PIN, LOW);
  } else {
    delay(500);
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **two Relays**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relays to **GP10/GP11**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the weight potentiometer to simulate package weights and check if the correct logs print over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Weight_g,Sort_code
0.00,0
50.00,1
250.00,2
```

## Expected Canvas Behavior
* Empty Scale (< 10g): OLED reads `CLASS: EMPTY`. Both relays are OFF.
* Underweight Package (50g): OLED reads `CLASS: UNDERWEIGHT`. Relay 1 turns ON. Bluetooth prints `50.00,1`.
* Overweight Package (250g): OLED reads `CLASS: OVERWEIGHT`. Relay 2 turns ON. Bluetooth prints `250.00,2`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(sortCode)` | Transmits the sorting result code (0 = Standard/Empty, 1 = Underweight, 2 = Overweight) over the Bluetooth UART channel. |

## Hardware & Safety Concept: Industrial Sorting Gates
Industrial sorting systems route packages to different bins (such as sorting by size, weight, or ZIP code). Pneumatic reject arms or diverter gates open quickly to guide items into their correct channels. Emergency limit monitoring is critical: if a gate jams, safety sensors shut down the conveyor line to prevent package blockages.

## Try This! (Challenges)
1. **Auditory Alert**: Connect a buzzer on GP12 and sound a beep when a package is sorted.
2. **Interactive Calibration**: Add a push button on GP16 that zeroes the scale when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reject gates trigger on empty scale | Zero calibration drift | Ensure the scale tares correctly on boot. You can also increase the empty scale detection threshold in code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [135 - Pico Industrial Scale](../advanced/135-pico-industrial-scale.md)
- [154 - Pico Industrial Scale OLED](../advanced/154-pico-industrial-scale-oled.md)
- [165 - Pico Industrial Scale Sorter](../advanced/165-pico-industrial-scale-sorter.md)
