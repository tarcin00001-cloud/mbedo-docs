# 159 - Pico Industrial Scale EEPROM

Build an industrial weighing scale that calibrates its zero offset (tare) dynamically and stores the calibration factor in non-volatile flash memory (EEPROM).

## Goal
Learn how to use the `EEPROM` library to store high-precision float settings, read HX711 ADCs, and tare values using physical buttons.

## What You Will Build
A calibratable digital weighing station:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Measures weights in grams.
- **Tare Push Button (GP16)**: Zeroes the scale weight offset when pressed.
- **EEPROM (Non-volatile Flash)**: Stores the zero offset value at address `20`, ensuring calibration persists across power reboots.
- **16x2 I2C LCD (GP4, GP5)**: Displays active weight and calibration statuses.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Push Button | `button` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT / PD_SCK | GP14 / GP15 | Sensor lines |
| Push Button | Terminal 1 | GP16 | Tare button input (active-low) |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <EEPROM.h>
#include "HX711.h"

const int DOUT_PIN = 14;
const int SCK_PIN  = 15;
const int TARE_PIN = 16;

HX711 scale;
LiquidCrystal_I2C lcd(0x27, 16, 2);

float calibration_factor = 420.0;
long zeroOffset = 0;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(TARE_PIN, INPUT_PULLUP);

  scale.begin(DOUT_PIN, SCK_PIN);
  scale.set_scale(calibration_factor);

  EEPROM.begin(512);

  // Read stored zeroOffset from EEPROM address 20-23
  byte b0 = EEPROM.read(20);
  byte b1 = EEPROM.read(21);
  byte b2 = EEPROM.read(22);
  byte b3 = EEPROM.read(23);

  // Reassemble long from 4 bytes
  zeroOffset = ((long)b0 << 24) | ((long)b1 << 16) | ((long)b2 << 8) | b3;

  // Verify if it is valid data (not empty flash 0xFFFFFFFF)
  if (zeroOffset == -1 || zeroOffset == 0xFFFFFFFF) {
    zeroOffset = 0;
  }

  scale.set_offset(zeroOffset);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Scale Station");
  lcd.setCursor(0, 1);
  lcd.print("Calibrating...  ");
  delay(1500);
}

void loop() {
  // Read tare button press
  if (digitalRead(TARE_PIN) == LOW) {
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Zeroing scale...");
    delay(1000); // Settle delay

    scale.tare(); // Calibrate zero offset
    zeroOffset = scale.get_offset();

    // Split long into 4 bytes
    byte b0 = (zeroOffset >> 24) & 0xFF;
    byte b1 = (zeroOffset >> 16) & 0xFF;
    byte b2 = (zeroOffset >> 8) & 0xFF;
    byte b3 = zeroOffset & 0xFF;

    // Save to EEPROM
    EEPROM.write(20, b0);
    EEPROM.write(21, b1);
    EEPROM.write(22, b2);
    EEPROM.write(23, b3);
    EEPROM.commit();

    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Offset Saved!");
    lcd.setCursor(0, 1);
    lcd.print(zeroOffset);
    delay(2000);
  }

  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Weight: ");
  lcd.print(weight, 1);
  lcd.print(" g");

  lcd.setCursor(0, 1);
  lcd.print("Offset: ");
  lcd.print(zeroOffset);

  delay(400); // Update twice per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **Push Button**, and **I2C LCD** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Button to **GP16**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weight potentiometer, click the Tare button on canvas, and verify that the zero offset updates and stays saved in memory when you restart.

## Expected Output

Terminal:
```
Simulation active. Zero calibration persistent engine online.
```

## Expected Canvas Behavior
* Startup: LCD reads `Weight: 0.0 g` (loads previous offset).
* Add weight (Slide pot to 150g): LCD reads `Weight: 150.0 g`.
* Press Tare button: LCD flashes `Zeroing scale...`, then zeroes out. `Weight` reads `0.0 g`, `Offset` updates to new value.
* Restart Simulation: Scale boots up with the newly tared offset loaded from memory.

## Code Walkthrough
| Expression | Expression Meaning |
| --- | --- |
| `((long)b0 << 24) \| ...` | Reassembles a 32-bit `long` zero offset value from 4 independent bytes read from EEPROM. |

## Hardware & Safety Concept: Sensor Calibration Persistence
Industrial weighing systems experience structural wear and environmental drift over time. Tare buttons let operators zero out the weight of containers or dust accumulation. Storing these calibration offsets in non-volatile flash memory ensures that the scale does not need to be recalibrated every time it is turned off.

## Try This! (Challenges)
1. **Calibration Scale Factor**: Add a second button on GP17 that calibrates the scale factor dynamically when a standard 100g weight is placed on it.
2. **Audio Chime**: Sound a short beep on GP12 every time tare calibration is completed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Tare offset reads negative numbers | Noise on startup | Ensure the scale has settled and is completely empty before pressing the Tare button to get a clean zero reading. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [109 - Pico Load Cell LCD](../intermediate/109-pico-loadcell-lcd.md)
- [135 - Pico Industrial Scale](135-pico-industrial-scale.md)
- [156 - Pico RFID EEPROM](156-pico-rfid-eeprom.md)
