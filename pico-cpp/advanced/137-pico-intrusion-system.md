# 137 - Pico Intrusion System

Build a dual-beam laser tripwire security grid that latches an alarm relay on breach and logs status on an LCD.

## Goal
Learn how to control multiple light transmitters (lasers), read analog light receivers (LDRs) in parallel, and latch alarm relays on breach events.

## What You Will Build
A dual-beam security tripwire barrier:
- **Laser Transmitter 1 (GP16)**: Constantly ON.
- **Laser Transmitter 2 (GP17)**: Constantly ON.
- **LDR Receiver 1 (GP26)**: Aligned with Laser 1.
- **LDR Receiver 2 (GP27)**: Aligned with Laser 2.
- **Relay Module (GP10)**: Latches ON (powering a security siren) if either beam is broken.
- **Active Buzzer (GP14)**: Sounds warning beeps during alerts.
- **16x2 I2C LCD (GP4, GP5)**: Displays grid status and breach logs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diodes | `led` | Yes (represented by separate LEDs) | Yes (low-power laser pointers) |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser 1 / 2 | Anode | GP16 / GP17 | Laser power control |
| LDR 1 / 2 | Signal (Pin 2) | GP26 / GP27 | Analog light inputs (with 10k pull-down) |
| Relay Module | IN | GP10 | Siren switch control |
| Active Buzzer | VCC (+) | GP14 | Alarm chime pin |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int LASER1_PIN = 16;
const int LASER2_PIN = 17;
const int LDR1_PIN   = 26;
const int LDR2_PIN   = 27;
const int RELAY_PIN  = 10;
const int BUZZ_PIN   = 14;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Tripwire limit: light drops below threshold when laser beam is blocked
const int TRIP_LIMIT = 2000;

bool alarmLatched = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(LASER1_PIN, OUTPUT);
  pinMode(LASER2_PIN, OUTPUT);
  pinMode(LDR1_PIN, INPUT);
  pinMode(LDR2_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);

  // Turn ON both lasers
  digitalWrite(LASER1_PIN, HIGH);
  digitalWrite(LASER2_PIN, HIGH);

  // Initialize outputs
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);

  lcd.init();
  lcd.backlight();
  resetSecurityState();
}

void loop() {
  int ldr1 = analogRead(LDR1_PIN);
  int ldr2 = analogRead(LDR2_PIN);

  bool b1_broken = (ldr1 < TRIP_LIMIT);
  bool b2_broken = (ldr2 < TRIP_LIMIT);

  // Latch alarm if either beam is crossed
  if ((b1_broken || b2_broken) && !alarmLatched) {
    alarmLatched = true;
    digitalWrite(RELAY_PIN, HIGH); // Turn alarm siren ON
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("!! GRID BREACH !!");
    lcd.setCursor(0, 1);
    if (b1_broken && b2_broken) {
      lcd.print("Beams: 1 & 2 CUT");
    } else if (b1_broken) {
      lcd.print("Beam: 1 CUT     ");
    } else {
      lcd.print("Beam: 2 CUT     ");
    }
  }

  // Siren operation if alarm is active
  if (alarmLatched) {
    digitalWrite(BUZZ_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZ_PIN, LOW);
    delay(100);
  }

  delay(20);
}

void resetSecurityState() {
  alarmLatched = false;
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Laser Grid Sys");
  lcd.setCursor(0, 1);
  lcd.print("Grid: SECURE    ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Lasers** (represented by LEDs), **two LDRs**, **Relay**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Connect Lasers to **GP16/GP17**, LDRs to **GP26/GP27**, Relay to **GP10**, Buzzer to **GP14**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust either LDR slider down to break a laser beam and watch the alarm latch.

## Expected Output

Terminal:
```
Simulation active. Tripwire grid security system online.
```

## Expected Canvas Behavior
* Normal state: LCD reads `Grid: SECURE`. Relay is OFF.
* Beam 1 broken (LDR 1 < TRIP_LIMIT): LCD reads `!! GRID BREACH !!` / `Beam: 1 CUT`. Relay turns ON, buzzer beeps.
* Beam 2 broken (LDR 2 < TRIP_LIMIT): LCD reads `!! GRID BREACH !!` / `Beam: 2 CUT`. Relay turns ON, buzzer beeps.

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
- [131 - Pico Anti-Theft](131-pico-anti-theft.md)
