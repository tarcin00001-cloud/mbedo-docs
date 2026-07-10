# 173 - Pico Industrial Scale Datalogger

Build an industrial checkweigher conveyor station that weighs items on a load cell, activates a reject relay gate, and streams weight log strings over Bluetooth.

## Goal
Learn how to read high-precision load cells via the HX711, activate sorting relays, update LCD statuses, and stream telemetry CSV logs over Bluetooth UART channels.

## What You Will Build
A wireless industrial checkweigher logging station:
- **HX711 & Load Cell (DOUT GP14, SCK GP15)**: Weighs items in grams.
- **Relay Module (GP10)**: Actuates a pneumatic reject arm for off-spec items.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits CSV logs wirelessly.
- **16x2 I2C LCD (GP4, GP5)**: Displays active weight and quality classifications (PASS/REJECT).
- **Bluetooth CSV Logs**: Streams log coordinates (e.g. `Weight_g,Status_code`) every 2 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HX711 Amplifier Module | `hx711` | Yes | Yes |
| Load Cell Sensor | `potentiometer` | Yes (represented by potentiometer slider) | Yes |
| Relay Module | `relay` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HX711 | DOUT / PD_SCK | GP14 / GP15 | Load cell interface lines |
| Relay Module | IN | GP10 | Reject gate control |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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

// Sorter target boundaries (150g target +/- 15g tolerance)
const float TARGET_WEIGHT = 150.0;
const float TOLERANCE     = 15.0;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART
  
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
  lcd.print("Checkweigher sys");
  lcd.setCursor(0, 1);
  lcd.print("Logger Online   ");
  
  // Print CSV Header over Bluetooth
  Serial1.println("Weight_g,Status_code");
  delay(1500);
}

void loop() {
  // Read weight (average over 5 readings)
  float weight = scale.get_units(5);
  if (weight < 0) { weight = 0.0; }

  int statusCode = 0; // 0 = Empty, 1 = Pass, 2 = Reject
  char statusText[10] = "EMPTY";

  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Weight: ");
  lcd.print(weight, 1);
  lcd.print(" g");

  // Sorting and logging logic
  lcd.setCursor(0, 1);
  if (weight > 10.0) { // Check only if an item is on the scale
    float diff = weight - TARGET_WEIGHT;
    if (diff < 0) { diff = -diff; }

    if (diff <= TOLERANCE) {
      // PASS
      digitalWrite(RELAY_PIN, LOW);
      statusCode = 1;
      statusText[0] = 'P'; statusText[1] = 'A'; statusText[2] = 'S'; statusText[3] = 'S'; statusText[4] = '\0';
    } else {
      // REJECT
      digitalWrite(RELAY_PIN, HIGH);
      statusCode = 2;
      statusText[0] = 'R'; statusText[1] = 'E'; statusText[2] = 'J'; statusText[3] = 'E'; statusText[4] = 'C'; statusText[5] = 'T'; statusText[6] = '\0';
      
      lcd.print("Status: REJECT! ");
      delay(1500); // Hold reject arm
      digitalWrite(RELAY_PIN, LOW);
    }
  } else {
    digitalWrite(RELAY_PIN, LOW);
  }

  // Update status label on LCD
  if (statusCode != 2) {
    lcd.print("Status: ");
    lcd.print(statusText);
  }

  // Stream CSV logs over Bluetooth
  Serial1.print(weight, 2);
  Serial1.print(",");
  Serial1.println(statusCode);

  delay(2000); // Update once every 2 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HX711**, **Load Cell** (represented by potentiometer), **Relay**, **HC-05**, and **I2C LCD** onto the canvas.
2. Connect HX711 to **GP14/GP15**, Relay to **GP10**, HC-05 to **GP0/GP1**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the weight potentiometer to simulate package weights and check if the correct logs print over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Weight_g,Status_code
0.00,0
145.00,1
100.00,2
```

## Expected Canvas Behavior
* No Package (Weight < 10g): LCD reads `Status: EMPTY`. Relay is OFF. CSV prints `0.00,0`.
* Target Package (145g): LCD reads `Status: PASS`. Relay remains OFF. CSV prints `145.00,1`.
* Faulty Package (100g): LCD reads `Status: REJECT!`. Relay turns ON for 1.5 seconds. CSV prints `100.00,2`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(statusCode)` | Transmits the sorting result code (0 = Empty, 1 = Pass, 2 = Reject) wirelessly to the Bluetooth host. |

## Hardware & Safety Concept: Industrial Checkweighers
Industrial checkweighers verify package weights on high-speed conveyor lines (e.g. food packages or medicine containers) to prevent shipping underfilled boxes. If a package is underweight, a pneumatic cylinder reject arm acts immediately to push the bad package into a separate reject bin without stopping the main conveyor line.

## Try This! (Challenges)
1. **Auditory Alarm**: Connect a buzzer on GP12 and play an alarm beep when a package is rejected.
2. **Interactive Tare**: Add a push button on GP16 that zeroes the scale when pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reject arm triggers on empty scale | Zero calibration drift | Ensure the scale tares correctly on boot. You can also increase the empty scale detection threshold in code. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [109 - Pico Load Cell LCD](../intermediate/109-pico-loadcell-lcd.md)
- [135 - Pico Industrial Scale](../advanced/135-pico-industrial-scale.md)
- [159 - Pico Industrial Scale EEPROM](../advanced/159-pico-industrial-scale-eeprom.md)
