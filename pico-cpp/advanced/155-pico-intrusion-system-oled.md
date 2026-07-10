# 155 - Pico Intrusion System OLED

Build an advanced dual-beam laser security grid that displays breach coordinates on an OLED and latches a warning relay.

## Goal
Learn how to control multiple light transmitters (lasers), read analog LDR receivers in parallel, latch alert states, and design graphic breach HUDs on SSD1306 displays.

## What You Will Build
A dual-beam laser security grid:
- **Laser Transmitter 1 (GP16)**: Constantly ON.
- **Laser Transmitter 2 (GP17)**: Constantly ON.
- **LDR Receiver 1 (GP26)**: Monitors Laser 1 beam.
- **LDR Receiver 2 (GP27)**: Monitors Laser 2 beam.
- **Relay Module (GP10)**: Latches ON (powering a security siren) when a breach occurs.
- **SSD1306 OLED (GP4, GP5)**: Displays active beam status lines and logs which specific barrier was cut.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diodes | `led` | Yes (represented by separate LEDs) | Yes (low-power laser pointers) |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser 1 / 2 | Anode | GP16 / GP17 | Laser power control |
| LDR 1 / 2 | Signal (Pin 2) | GP26 / GP27 | Analog inputs (requires 10k pull-downs) |
| Relay Module | IN | GP10 | Security siren control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SSD1306.h>

const int LASER1_PIN = 16;
const int LASER2_PIN = 17;
const int LDR1_PIN   = 26;
const int LDR2_PIN   = 27;
const int RELAY_PIN  = 10;

#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64
#define OLED_RESET    -1
Adafruit_SSD1306 display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, OLED_RESET);

// Light threshold level (drops low when beam is broken)
const int TRIP_LIMIT = 2000;

bool alarmLatched = false;
char breachMessage[20] = "GRID: SECURE";

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(LASER1_PIN, OUTPUT);
  pinMode(LASER2_PIN, OUTPUT);
  pinMode(LDR1_PIN, INPUT);
  pinMode(LDR2_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);

  // Turn ON lasers
  digitalWrite(LASER1_PIN, HIGH);
  digitalWrite(LASER2_PIN, HIGH);

  digitalWrite(RELAY_PIN, LOW); // Siren OFF

  if (display.begin(SSD1306_SWITCHCAPVCC, 0x3C)) {
    display.clearDisplay();
    display.display();
  }
}

void loop() {
  int ldr1 = analogRead(LDR1_PIN);
  int ldr2 = analogRead(LDR2_PIN);

  bool b1_broken = (ldr1 < TRIP_LIMIT);
  bool b2_broken = (ldr2 < TRIP_LIMIT);

  // Latch alarm state on first breach
  if ((b1_broken || b2_broken) && !alarmLatched) {
    alarmLatched = true;
    digitalWrite(RELAY_PIN, HIGH); // Turn siren ON

    if (b1_broken && b2_broken) {
      breachMessage[0] = 'B'; breachMessage[1] = 'O'; breachMessage[2] = 'T'; breachMessage[3] = 'H'; breachMessage[4] = ' '; breachMessage[5] = 'C'; breachMessage[6] = 'U'; breachMessage[7] = 'T'; breachMessage[8] = '\0';
    } else if (b1_broken) {
      breachMessage[0] = 'B'; breachMessage[1] = 'E'; breachMessage[2] = 'A'; breachMessage[3] = 'M'; breachMessage[4] = ' '; breachMessage[5] = '1'; breachMessage[6] = ' '; breachMessage[7] = 'C'; breachMessage[8] = 'U'; breachMessage[9] = 'T'; breachMessage[10] = '\0';
    } else {
      breachMessage[0] = 'B'; breachMessage[1] = 'E'; breachMessage[2] = 'A'; breachMessage[3] = 'M'; breachMessage[4] = ' '; breachMessage[5] = '2'; breachMessage[6] = ' '; breachMessage[7] = 'C'; breachMessage[8] = 'U'; breachMessage[9] = 'T'; breachMessage[10] = '\0';
    }
  }

  // Draw HUD on OLED
  display.clearDisplay();

  if (alarmLatched) {
    // INTRUDER ALERT BREACH
    display.fillRect(0, 0, 128, 64, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setTextSize(2);
    display.setCursor(15, 10);
    display.print("BREACH!");
    
    display.setTextSize(1);
    display.setCursor(15, 42);
    display.print("Log: ");
    display.print(breachMessage);
  } else {
    // SECURE MONITORING
    display.fillRect(0, 0, 128, 14, SSD1306_WHITE);
    display.setTextColor(SSD1306_BLACK);
    display.setCursor(20, 3);
    display.print("LASER GRID HUD");

    display.setTextColor(SSD1306_WHITE);
    display.setTextSize(1);
    
    display.setCursor(10, 24);
    display.print("Beam 1: ");
    display.print(ldr1);
    
    display.setCursor(10, 40);
    display.print("Beam 2: ");
    display.print(ldr2);

    display.drawRect(0, 0, 128, 64, SSD1306_WHITE);
  }

  display.display();
  delay(100);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Lasers** (represented by LEDs), **two LDRs**, **Relay**, and **SSD1306 OLED** onto the canvas.
2. Connect Lasers to **GP16/GP17**, LDRs to **GP26/GP27**, Relay to **GP10**, and OLED to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust either LDR slider down to break a laser beam and watch the alarm latch and display on the OLED.

## Expected Output

Terminal:
```
Simulation active. Laser tripwire grid with OLED breach logs online.
```

## Expected Canvas Behavior
* Normal state: OLED reads `LASER GRID HUD` showing LDR sensor readings. Relay is OFF.
* Beam 1 broken (LDR 1 < TRIP_LIMIT): OLED screen fills white, flashing `BREACH!` / `Log: BEAM 1 CUT`. Relay turns ON (Siren ON) and stays latched.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `b1_broken \|\| b2_broken` | Logical check that triggers alarms if either beam is broken, creating a grid fence. |

## Hardware & Safety Concept: Grid Barrier Security
A single laser tripwire can be easily stepped over or crawled under. Multi-beam grids stack lasers at varying heights (e.g. 30 cm and 100 cm above the ground) to prevent intruders from bypassing the tripwire. For safety, industrial security systems route power through the relays so they automatically trip if power is cut or wires are severed.

## Try This! (Challenges)
1. **Disarm Override Key**: Connect a button on GP15 and configure it to reset the alarm and re-arm the grid.
2. **Breach Log**: Send "ALERT: BEAM 1 BROKEN" messages to the Serial Monitor.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Laser misalignment | Ensure the laser pointers are pointing directly at the LDR sensors. Even a minor bump to the sensor frame can cause misalignments. |

## Mode Notes
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [108 - Pico Intrusion Alarm](../intermediate/108-pico-intrusion-alarm.md)
- [121 - Pico Fire System](121-pico-fire-system.md)
- [137 - Pico Intrusion System](137-pico-intrusion-system.md)
