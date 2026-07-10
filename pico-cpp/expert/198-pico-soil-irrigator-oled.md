# 198 - Pico Soil Irrigator OLED

Build a dual-zone automated agricultural irrigator station that waters dry soils and displays moisture metrics and valve statuses on an OLED screen.

## Goal
Learn how to read multiple analog moisture inputs, control separate high-load relays, and design structured layout dashboards on SSD1306 OLED displays.

## What You Will Build
A dual-zone smart irrigation controller:
- **Soil Moisture Sensor 1 (GP26)**: Monitors Zone 1 soil.
- **Soil Moisture Sensor 2 (GP27)**: Monitors Zone 2 soil.
- **Relay Valve 1 (GP10)**: Waters Zone 1 when dry.
- **Relay Valve 2 (GP11)**: Waters Zone 2 when dry.
- **SSD1306 OLED (GP4, GP5)**: Displays moisture indexes and valve states for both zones.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Soil Moisture Sensors | `potentiometer` | Yes (two potentiometers) | Yes |
| Relay Modules | `relay` | Yes (two relays) | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Soil Sensor 1 | AO | GP26 | Zone 1 moisture |
| Soil Sensor 2 | AO | GP27 | Zone 2 moisture |
| Relay 1 | IN | GP10 | Zone 1 valve |
| Relay 2 | IN | GP11 | Zone 2 valve |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int SOIL1_PIN  = 26;
const int SOIL2_PIN  = 27;
const int RELAY1_PIN = 10;
const int RELAY2_PIN = 11;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Moisture threshold (high values indicate dry soil)
const int DRY_LIMIT = 2800;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(SOIL1_PIN, INPUT);
  pinMode(SOIL2_PIN, INPUT);
  pinMode(RELAY1_PIN, OUTPUT);
  pinMode(RELAY2_PIN, OUTPUT);

  digitalWrite(RELAY1_PIN, LOW); // Close valves
  digitalWrite(RELAY2_PIN, LOW);

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
  delay(1000);
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

  // Update OLED Screen
  display.clearDisplay();

  // Draw header block
  display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
  display.setTextColor(SSD1306_BLACK);
  display.setCursor(20, 3);
  display.print("IRRIGATION HUD");

  display.setTextColor(SSD1306_WHITE);
  display.setTextSize(1);

  // Row 1: Zone 1 status
  display.setCursor(8, 20);
  display.print("Z1: ");
  display.print(m1);
  display.print(v1_active ? " (WATERING)" : " (STANDBY)");

  // Row 2: Zone 2 status
  display.setCursor(8, 36);
  display.print("Z2: ");
  display.print(m2);
  display.print(v2_active ? " (WATERING)" : " (STANDBY)");

  display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  display.display();

  delay(2000); // Soak delay
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Soil Sensors** (represented by potentiometers), **two Relays**, and **SSD1306 OLED** onto the canvas.
2. Connect Soil 1 to **GP26**, Soil 2 to **GP27**, Relay 1 to **GP10**, Relay 2 to **GP11**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the soil sensors to dry, and check if the valves open and if the OLED dashboard updates.

## Expected Output

Terminal:
```
Simulation active. Dual zone irrigation OLED engine active.
```

## Expected Canvas Behavior
* Normal state (Moist soil): Relays are OFF. OLED reads `(STANDBY)`.
* Soil 1 Dry (> 2800): Relay 1 turns ON (Valve 1 opens). OLED Zone 1 line reads `Z1: 3200 (WATERING)`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY1_PIN, HIGH)` | Drives GP10 HIGH to open the first zone's irrigation valve. |

## Hardware & Safety Concept: Multi-Channel Solenoid Valve Wiring
When wiring multiple solenoids (relays) to a single power block, the current draw spikes when multiple valves open at the same time. To protect the power supply, systems use **flyback diodes** (1N4007) wired in reverse across each relay coil. These diodes absorb the high-voltage spike generated when the coil is turned OFF, protecting the microcontroller from damage.

## Try This! (Challenges)
1. **Critical Warning Tone**: Connect a buzzer on GP14 and sound an alarm beep if either zone remains dry for too long.
2. **Alternate watering scheduler**: Add logic to ensure only one zone can water at a time to keep water pressure high.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| OLED screen stays stuck blank | I2C address conflict | Verify that the display SDA/SCL lines are shared correctly on GP4/GP5 in parallel. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [110 - Pico Irrigation Station](../intermediate/110-pico-irrigation-station.md)
- [122 - Pico Greenhouse](../advanced/122-pico-greenhouse.md)
- [130 - Pico Soil Irrigator](../advanced/130-pico-soil-irrigator.md)
- [188 - Pico Soil Irrigator Logger](188-pico-soil-irrigator-logger.md)
