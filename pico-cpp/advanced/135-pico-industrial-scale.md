# 135 - Pico Industrial Scale

Build an industrial checkweigher sorting gate that weighs items via a load cell and activates a relay reject gate if weight is out of limits.

## Goal
Learn how to interface high-precision ADC modules (HX711), display real-time weight values on an LCD, and trigger sorting relay gates based on weight thresholds.

## What You Will Build
An automated conveyor checkweigher:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures item weight in grams.
- **Relay Module (GP10)**: Actuates a pneumatic reject arm to push underweight items off the conveyor belt.
- **16x2 I2C LCD (GP4, GP5)**: Displays the item weight and sorting classification (e.g. "Weight: 150g | PASS" or "REJECT").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT | GP14 | Serial data output |
| HX711 | PD_SCK | GP15 | Serial clock input |
| Relay Module | IN | GP10 | Pneumatic sorting arm |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include "HX711.h"

const int DOUT_PIN  = 14;
const int SCK_PIN   = 15;
const int RELAY_PIN = 10;

HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

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

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Checkweigher");
  lcd.setCursor(0, 1);
  lcd.print("Scale Ready     ");
  delay(1500);
}

void loop() {
  // Read weight (average over 5 readings)
  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Wt: ");
  lcd.print(weight, 1);
  lcd.print(" g");

  // Sorting logic: check if weight is within tolerance limits
  lcd.setCursor(0, 1);
  if (weight > 10.0) { // Check only if an item is physically on the scale
    float diff = weight - TARGET_WEIGHT;
    if (diff < 0) { diff = -diff; }

    if (diff <= TOLERANCE) {
      // Weight matches specification - allow passage
      digitalWrite(RELAY_PIN, LOW);
      lcd.print("Status: PASS    ");
    } else {
      // Under/Overweight item - activate reject valve
      digitalWrite(RELAY_PIN, HIGH);
      lcd.print("Status: REJECT! ");
      delay(1500); // Keep reject arm extended to clear item
      digitalWrite(RELAY_PIN, LOW);
    }
  } else {
    // Scale empty
    digitalWrite(RELAY_PIN, LOW);
    lcd.print("Status: EMPTY   ");
  }

  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **Relay**, and **I2C LCD** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relay to **GP10**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weight potentiometer to simulate package weights and check if the sorting relay triggers.

## Expected Output

Terminal:
```
Simulation active. Checkweigher online. Zero calibration complete.
```

## Expected Canvas Behavior
* No Package (Weight < 10g): LCD reads `Status: EMPTY`. Relay is OFF.
* Target Package (Weight 145g): LCD reads `Wt: 145.0 g` / `Status: PASS`. Relay remains OFF.
* Faulty Package (Weight 100g): LCD reads `Wt: 100.0 g` / `Status: REJECT!`. Relay turns ON (Reject arm active) for 1.5 seconds.

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
| Reject arm triggers on empty scale | Zero calibration drift | Ensure the scale tares correctly on boot. You can also increase the empty scale detection threshold in code from `10.0` to `20.0`. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [109 - Pico Load Cell LCD](../intermediate/109-pico-loadcell-lcd.md)
