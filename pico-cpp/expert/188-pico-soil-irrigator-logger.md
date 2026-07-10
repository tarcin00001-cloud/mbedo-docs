# 188 - Pico Soil Irrigator Logger

Build a dual-zone smart agricultural irrigator station that water dry soils, displays status on an LCD, and streams event logs over Bluetooth.

## Goal
Learn how to read multiple analog sensors, control dual high-load relays, update text screens, and stream wireless telemetry logs over Bluetooth UART channels.

## What You Will Build
A dual-zone agricultural irrigation controller:
- **Soil Moisture Sensor 1 (GP26)**: Monitors Zone 1 soil.
- **Soil Moisture Sensor 2 (GP27)**: Monitors Zone 2 soil.
- **Relay Valve 1 (GP10)**: Waters Zone 1 when dry.
- **Relay Valve 2 (GP11)**: Waters Zone 2 when dry.
- **16x2 I2C LCD (GP4, GP5)**: Displays moisture levels and valve status.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless event logs (e.g., `Z1_ON,Z2_ON`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensors | `potentiometer` | Yes (two potentiometers) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Soil Sensor 1 | AO | GP26 | Zone 1 moisture |
| Soil Sensor 2 | AO | GP27 | Zone 2 moisture |
| Relay 1 | IN | GP10 | Zone 1 valve |
| Relay 2 | IN | GP11 | Zone 2 valve |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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

// Moisture threshold (high values indicate dry soil)
const int DRY_LIMIT = 2800;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(SOIL1_PIN, INPUT);
  pinMode(SOIL2_PIN, INPUT);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);

  digitalWrite(RELAY1_PIN, LOW); // Close valves
  digitalWrite(RELAY2_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Soil Sorter Node");
  lcd.setCursor(0, 1);
  lcd.print("Logger Online   ");

  // Print CSV Header to Bluetooth
  Serial1.println("Soil1,Soil2,Valve1_ON,Valve2_ON");
  delay(1500);
}

void loop() {
  int m1 = analogRead(SOIL1_PIN);
  int m2 = analogRead(SOIL2_PIN);

  bool v1_active = false;
  bool v2_active = false;

  // Zone 1 irrigation logic
  if (m1 > DRY_LIMIT) {
    digitalWrite(RELAY1_PIN, HIGH); // Open Zone 1 Valve
    v1_active = true;
  } else {
    digitalWrite(RELAY1_PIN, LOW);
  }

  // Zone 2 irrigation logic
  if (m2 > DRY_LIMIT) {
    digitalWrite(RELAY2_PIN, HIGH); // Open Zone 2 Valve
    v2_active = true;
  } else {
    digitalWrite(RELAY2_PIN, LOW);
  }

  // Update LCD Screen
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Z1: ");
  lcd.print(m1);
  lcd.print(v1_active ? " WTR" : " Off");

  lcd.setCursor(0, 1);
  lcd.print("Z2: ");
  lcd.print(m2);
  lcd.print(v2_active ? " WTR" : " Off");

  // Stream CSV logs over Bluetooth
  Serial1.print(m1);
  Serial1.print(",");
  Serial1.print(m2);
  Serial1.print(",");
  Serial1.print(v1_active ? 1 : 0);
  Serial1.print(",");
  Serial1.println(v2_active ? 1 : 0);

  delay(3000); // Check and log once every 3 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Soil Sensors** (represented by potentiometers), **two Relays**, **I2C LCD**, and **HC-05** onto the canvas.
2. Connect sensors, relays, LCD, and Bluetooth.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the soil sensors to dry, and check if the valves open and if the logs print over Bluetooth.

## Expected Output

Terminal (Bluetooth):
```
Soil1,Soil2,Valve1_ON,Valve2_ON
1200,1300,0,0
3200,1300,1,0
```

## Expected Canvas Behavior
* Normal state (Moist soil): Relays are OFF. LCD reads `Off`.
* Soil 1 Dry (> 2800): Relay 1 turns ON (Valve 1 opens). LCD reads `Z1: 3200 WTR`. Bluetooth streams `3200,1300,1,0`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println(v2_active ? 1 : 0)` | Sends the Zone 2 valve state (1 = Open/Watering, 0 = Closed) over the Bluetooth link. |

## Hardware & Safety Concept: Multi-Channel Solenoid Valve Wiring
When wiring multiple solenoids (relays) to a single power block, the current draw spikes when multiple valves open at the same time. To protect the power supply, systems use **flyback diodes** (1N4007) wired in reverse across each relay coil. These diodes absorb the high-voltage spike generated when the coil is turned OFF, protecting the microcontroller from damage.

## Try This! (Challenges)
1. **Critical High Alarm**: Connect a buzzer on GP14 and sound an alarm beep if either zone remains dry for too long.
2. **Alternate watering scheduler**: Add logic to ensure only one zone can water at a time to keep water pressure high.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD flashes repeatedly | Screen update rate too high | Confirm the loop includes a slow 3-second check delay (`delay(3000)`) to let values settle. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [110 - Pico Irrigation Station](../intermediate/110-pico-irrigation-station.md)
- [122 - Pico Greenhouse](../advanced/122-pico-greenhouse.md)
- [130 - Pico Soil Irrigator](../advanced/130-pico-soil-irrigator.md)
