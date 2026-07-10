# 166 - Pico Intrusion Datalogger

Build an advanced dual-beam laser security grid that displays status on an LCD and streams real-time breach log strings over Bluetooth.

## Goal
Learn how to control light transmitters (lasers), read analog light sensors (LDRs) in parallel, and transmit real-time ASCII security warnings over a Bluetooth serial link (UART).

## What You Will Build
A wireless security barrier:
- **Laser Transmitter 1 (GP16)**: Constantly ON.
- **Laser Transmitter 2 (GP17)**: Constantly ON.
- **LDR Receiver 1 (GP26)**: Aligned with Laser 1.
- **LDR Receiver 2 (GP27)**: Aligned with Laser 2.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits breach alerts wirelessly to a smartphone or computer.
- **16x2 I2C LCD (GP4, GP5)**: Displays active barrier grid status.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diodes | `led` | Yes (represented by separate LEDs) | Yes (low-power laser pointers) |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser 1 / 2 | Anode | GP16 / GP17 | Laser power control |
| LDR 1 / 2 | Signal (Pin 2) | GP26 / GP27 | Analog inputs (requires 10k pull-downs) |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Light threshold level (drops low when beam is broken)
const int TRIP_LIMIT = 2000;

bool alarmLatched = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  pinMode(LASER1_PIN, OUTPUT);
  pinMode(LASER2_PIN, OUTPUT);
  pinMode(LDR1_PIN, INPUT);
  pinMode(LDR2_PIN, INPUT);

  // Turn ON both lasers
  digitalWrite(LASER1_PIN, HIGH);
  digitalWrite(LASER2_PIN, HIGH);

  lcd.init();
  lcd.backlight();
  resetSecurityState();

  Serial1.println("Laser Security Grid Active.");
}

void loop() {
  int ldr1 = analogRead(LDR1_PIN);
  int ldr2 = analogRead(LDR2_PIN);

  bool b1_broken = (ldr1 < TRIP_LIMIT);
  bool b2_broken = (ldr2 < TRIP_LIMIT);

  // Latch alarm if either beam is broken
  if ((b1_broken || b2_broken) && !alarmLatched) {
    alarmLatched = true;
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("!! GRID BREACH !!");
    
    lcd.setCursor(0, 1);
    if (b1_broken && b2_broken) {
      lcd.print("Beams: 1 & 2 CUT");
      Serial1.println("ALERT: GRID BREACH - BEAMS 1 & 2 CUT");
    } else if (b1_broken) {
      lcd.print("Beam: 1 CUT     ");
      Serial1.println("ALERT: GRID BREACH - BEAM 1 CUT");
    } else {
      lcd.print("Beam: 2 CUT     ");
      Serial1.println("ALERT: GRID BREACH - BEAM 2 CUT");
    }
  }

  // Check for disarm reset command over Bluetooth
  if (Serial1.available()) {
    char cmd = Serial1.read();
    if (cmd == 'R') {
      resetSecurityState();
      Serial1.println("SYSTEM RESET: Grid Armed & Secure");
    }
  }

  delay(50);
}

void resetSecurityState() {
  alarmLatched = false;
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Laser Grid Sys");
  lcd.setCursor(0, 1);
  lcd.print("Grid: SECURE    ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two Lasers** (represented by LEDs), **two LDRs**, **HC-05**, and **I2C LCD** onto the canvas.
2. Connect Lasers to **GP16/GP17**, LDRs to **GP26/GP27**, HC-05 to **GP0/GP1**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust either LDR slider down to break a laser beam, check if the alert prints over Bluetooth, then type `'R'` to reset the system.

## Expected Output

Terminal (Bluetooth):
```
Laser Security Grid Active.
ALERT: GRID BREACH - BEAM 1 CUT
SYSTEM RESET: Grid Armed & Secure
```

## Expected Canvas Behavior
* Normal state: LCD reads `Grid: SECURE`. Silent.
* Beam 1 broken (LDR 1 < TRIP_LIMIT): LCD reads `!! GRID BREACH !!` / `Beam: 1 CUT`. Bluetooth transmits warning.
* Type `'R'` on BT: System resets, LCD returns to `Grid: SECURE`.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Serial1.println("ALERT: ...")` | Sends a real-time breach log message over the Bluetooth UART channel to the remote supervisor terminal. |

## Hardware & Safety Concept: Remote Security Logging
Wireless security sensors stream alerts over Bluetooth or Wi-Fi to a remote monitoring computer. To prevent intruders from jamming the wireless signal, security consoles use **heartbeat check-ins** (e.g. sending a "SYSTEM OK" status byte every 5 seconds). If the receiver misses a heartbeat check-in, it triggers an alert immediately.

## Try This! (Challenges)
1. **Physical Reset Button**: Add a push button on GP14 to reset and re-arm the grid.
2. **Alert Siren**: Connect a buzzer on GP15 and sound an alarm beep when a breach occurs.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Grid triggers on minor ambient light changes | Threshold set too high | Laser beams are bright. Ensure the `TRIP_LIMIT` is set lower than the laser's active light reading but higher than normal room lighting. |

## Mode Notes
This multi-device I2C and UART project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [108 - Pico Intrusion Alarm](../intermediate/108-pico-intrusion-alarm.md)
- [137 - Pico Intrusion System](137-pico-intrusion-system.md)
- [155 - Pico Intrusion System OLED](155-pico-intrusion-system-oled.md)
