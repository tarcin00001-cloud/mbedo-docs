# 109 - Pico Load Cell LCD

Build a digital scale that displays weight measurements from a load cell on a 16x2 LCD using an HX711 amplifier.

## Goal
Learn how to interface custom high-precision ADC modules (HX711), apply calibration factors, and print weight telemetry on LCD screens.

## What You Will Build
A digital kitchen scale:
- **HX711 Amplifier & Load Cell (DOUT GP14, SCK GP15)**: Measures mechanical weight force.
- **16x2 I2C LCD (GP4, GP5)**: Displays the calculated weight in grams (e.g. "Weight: 450 g").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor (5kg) | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | VCC | 3.3V | Power supply |
| HX711 | DOUT | GP14 | Serial data output |
| HX711 | PD_SCK | GP15 | Serial clock input |
| HX711 | GND | GND | Ground return |
| I2C LCD | VCC | 5V | LCD Power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "HX711.h"

const int DOUT_PIN = 14;
const int SCK_PIN  = 15;

HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Calibration factor (obtained through manual calibration testing)
float calibration_factor = 420.0; 

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  scale.begin(DOUT_PIN, SCK_PIN);
  scale.set_scale(calibration_factor);
  scale.tare(); // Zero the scale on boot (assumes no weight on tray)

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Digital Scale");
  delay(1000);
}

void loop() {
  // Read scaled weight (average over 5 readings)
  float weight = scale.get_units(5);

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Kitchen Scale");

  lcd.setCursor(0, 1);
  if (weight < 0) {
    weight = 0.0; // Filter minor negative fluctuations near zero
  }
  lcd.print("Weight: ");
  lcd.print(weight, 1); // Print with 1 decimal place
  lcd.print(" g");

  delay(500); // Update twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), and **I2C LCD** onto the canvas.
2. Connect HX711 DOUT to **GP14**, SCK to **GP15**. Connect LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weight force potentiometer on canvas to simulate adding items and watch the LCD weight update.

## Expected Output

Terminal:
```
Simulation active. Zero calibration complete. Scale ready.
```

## Expected Canvas Behavior
* Bootup: Zeroes weight value (displays `Weight: 0.0 g`).
* Weight slider moved up: Displays proportional weight in grams (e.g. `Weight: 350.5 g`).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `scale.tare()` | Resets the baseline weight calculation to zero. Any weight on the scale during boot is ignored (e.g. ignoring the weight of a measuring bowl). |
| `scale.get_units(5)` | Reads 5 raw weight samples from the HX711, averages them, applies the calibration factor, and returns the weight in grams. |

## Hardware & Safety Concept: Load Cell Strain Gauges
Load cells contain strain gauges wired in a **Wheatstone bridge** circuit. When weight is placed on the scale, the metal bar bends slightly (microscopic strain), causing a tiny change in electrical resistance. The HX711 is a high-precision **24-bit analog-to-digital converter (ADC)** designed to amplify this extremely small voltage change (microvolts) so the microcontroller can read it.

## Try This! (Challenges)
1. **Tare Button**: Connect a push button on GP16. When pressed, trigger `scale.tare()` to zero the scale manually.
2. **Overweight Warning**: Sound a buzzer on GP10 if the weight exceeds 2000 grams.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Weight values drift or fluctuate constantly | Mechanical vibrations | Ensure the load cell is bolted securely to a stable base plate. Vibrations or airflow can cause readings to fluctuate. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [26 - Pico Button Counter](../../beginner/26-pico-button-counter.md)
- [58 - Pico LCD Print](58-pico-lcd-print.md)
