# 130 - Pico Soil Irrigator

Build an automated agricultural control station that manages two independent soil zones with dedicated solenoid valves and status LCD updates.

## Goal
Learn how to read multiple analog soil moisture sensors, actuate separate relay channels, and display independent status readouts on I2C screen grids.

## What You Will Build
A dual-zone smart irrigation controller:
- **Soil Moisture Sensor 1 (GP26)**: Monitors Zone 1 soil.
- **Soil Moisture Sensor 2 (GP27)**: Monitors Zone 2 soil.
- **Relay Valve 1 (GP10)**: Waters Zone 1 when dry.
- **Relay Valve 2 (GP11)**: Waters Zone 2 when dry.
- **16x2 I2C LCD**: Displays moisture indices and valve states for both zones.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensors | `potentiometer` | Yes (two potentiometers) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Soil Sensor 1 | AO | GP26 | Zone 1 moisture |
| Soil Sensor 2 | AO | GP27 | Zone 2 moisture |
| Relay 1 | IN | GP10 | Zone 1 valve |
| Relay 2 | IN | GP11 | Zone 2 valve |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int SOIL1_PIN  = 26;
const int SOIL2_PIN  = 27;
const int RELAY1_PIN = 10;
const int RELAY2_PIN = 11;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Moisture thresholds (raw ADC)
// High values indicate dry soil.
const int DRY_LIMIT = 2800;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(SOIL1_PIN, INPUT);
  pinMode(SOIL2_PIN, INPUT);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);

  digitalWrite(RELAY1_PIN, LOW); // Start valves closed
  digitalWrite(RELAY2_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Dual Irrigation");
  lcd.setCursor(0, 1);
  lcd.print("Station Online  ");
  delay(1500);
}

void loop() {
  int m1 = analogRead(SOIL1_PIN);
  int m2 = analogRead(SOIL2_PIN);

  lcd.clear();

  // Zone 1 irrigation logic
  lcd.setCursor(0, 0);
  if (m1 > DRY_LIMIT) {
    digitalWrite(RELAY1_PIN, HIGH); // Open Zone 1 Valve
    lcd.print("Z1: DRY  | WTR");
  } else {
    digitalWrite(RELAY1_PIN, LOW);  // Close Zone 1 Valve
    lcd.print("Z1: OK   | Off");
  }

  // Zone 2 irrigation logic
  lcd.setCursor(0, 1);
  if (m2 > DRY_LIMIT) {
    digitalWrite(RELAY2_PIN, HIGH); // Open Zone 2 Valve
    lcd.print("Z2: DRY  | WTR");
  } else {
    digitalWrite(RELAY2_PIN, LOW);  // Close Zone 2 Valve
    lcd.print("Z2: OK   | Off");
  }

  // Soak / settlement wait delay
  delay(2000); 
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Soil Sensors** (represented by potentiometers), **two Relays**, and **I2C LCD** onto the canvas.
2. Connect Soil 1 to **GP26**, Soil 2 to **GP27**, Relay 1 to **GP10**, Relay 2 to **GP11**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide either sensor to dry (high values) and watch the respective relay activate and the LCD update.

## Expected Output

Terminal:
```
Simulation active. Dual zone irrigation logic active.
```

## Expected Canvas Behavior
* Soil 1 Dry, Soil 2 Wet: Row 0 reads `Z1: DRY  | WTR` (Relay 1 ON), Row 1 reads `Z2: OK   | Off` (Relay 2 OFF).
* Both Wet: Row 0 reads `Z1: OK   | Off`, Row 1 reads `Z2: OK   | Off` (Both relays OFF).

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY1_PIN, HIGH)` | Drives GP10 HIGH to open the first zone's irrigation valve. |

## Hardware & Safety Concept: Multi-Channel Solenoid Valve Wiring
When wiring multiple solenoids (relays) to a single power block, the current draw spikes when multiple valves open at the same time. To protect the power supply, systems use **flyback diodes** (1N4007) wired in reverse across each relay coil. These diodes absorb the high-voltage spike generated when the coil is turned OFF, protecting the microcontroller from damage.

## Try This! (Challenges)
1. **Critical Warning Tone**: Connect a buzzer on GP14 and sound a beep if either zone remains dry for too long.
2. **Alternate watering scheduler**: Add logic to ensure only one zone can water at a time to keep water pressure high.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes on and off constantly | Screen clear rate too high | Confirm the loop includes a slow 2-second check delay (`delay(2000)`) to let values settle. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [110 - Pico Irrigation Station](../intermediate/110-pico-irrigation-station.md)
- [122 - Pico Greenhouse](122-pico-greenhouse.md)
