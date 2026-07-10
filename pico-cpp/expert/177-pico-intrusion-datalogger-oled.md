# 177 - Pico Intrusion Datalogger OLED

Build an advanced dual-beam laser security grid that displays breach status on an OLED, latches a siren relay, and logs events wirelessly over Bluetooth.

## Goal
Learn how to control light transmitters (lasers), read analog LDR receivers in parallel, latch alarm relays, design graphic OLED layouts, and stream Bluetooth audit warnings.

## What You Will Build
A wireless security grid gateway:
- **Laser Transmitter 1 (GP16)**: Constantly ON.
- **Laser Transmitter 2 (GP17)**: Constantly ON.
- **LDR Receiver 1 (GP26)**: Monitors Laser 1 beam.
- **LDR Receiver 2 (GP27)**: Monitors Laser 2 beam.
- **Relay Module (GP10)**: Latches ON (powering a security siren) when a breach occurs.
- **SSD1306 OLED (GP4, GP5)**: Displays active beam status lines and logs which specific barrier was cut.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless breach warnings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Laser Diodes | `led` | Yes (represented by separate LEDs) | Yes (low-power laser pointers) |
| LDR Photoresistors | `ldr` | Yes (two LDRs) | Yes |
| Relay Module | `relay` | Yes | Yes |
| SSD1306 OLED (128x64) | `oled` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Laser 1 / 2 | Anode | GP16 / GP17 | Laser power control |
| LDR 1 / 2 | Signal (Pin 2) | GP26 / GP27 | Analog inputs (requires 10k pull-downs) |
| Relay Module | IN | GP10 | Security siren control |
| SSD1306 OLED | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
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

const int TRIP_LIMIT = 2000;

bool alarmLatched = false;
char breachMessage[20] = "GRID: SECURE";

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

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

  Serial1.println("Laser Security Grid Online.");
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
      Serial1.println("ALERT: GRID BREACH - BEAMS 1 & 2 CUT");
    } else if (b1_broken) {
      breachMessage[0] = 'B'; breachMessage[1] = 'E'; breachMessage[2] = 'A'; breachMessage[3] = 'M'; breachMessage[4] = ' '; breachMessage[5] = '1'; breachMessage[6] = ' '; breachMessage[7] = 'C'; breachMessage[8] = 'U'; breachMessage[9] = 'T'; breachMessage[10] = '\0';
      Serial1.println("ALERT: GRID BREACH - BEAM 1 CUT");
    } else {
      breachMessage[0] = 'B'; breachMessage[1] = 'E'; breachMessage[2] = 'A'; breachMessage[3] = 'M'; breachMessage[4] = ' '; breachMessage[5] = '2'; breachMessage[6] = ' '; breachMessage[7] = 'C'; breachMessage[8] = 'U'; breachMessage[9] = 'T'; breachMessage[10] = '\0';
      Serial1.println("ALERT: GRID BREACH - BEAM 2 CUT");
    }
  }

  // Check for disarm command over Bluetooth
  if (Serial1.available()) {
    char cmd = Serial1.read();
    if (cmd == 'R') {
      alarmLatched = false;
      digitalWrite(RELAY_PIN, LOW);
      breachMessage[0] = 'G'; breachMessage[1] = 'R'; breachMessage[2] = 'I'; breachMessage[3] = 'D'; breachMessage[4] = ':'; breachMessage[5] = ' '; breachMessage[6] = 'S'; breachMessage[7] = 'E'; breachMessage[8] = 'C'; breachMessage[9] = 'U'; breachMessage[10] = 'R'; breachMessage[11] = 'E'; breachMessage[12] = '\0';
      Serial1.println("SYSTEM RESET: Grid Armed & Secure");
    }
  }

  // Draw HUD on OLED
  display.clearDisplay();

  if (alarmLatched) {
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
1. Drag the **Raspberry Pi Pico**, **two Lasers** (represented by LEDs), **two LDRs**, **Relay**, **SSD1306 OLED**, and **HC-05** onto the canvas.
2. Connect Lasers to **GP16/GP17**, LDRs to **GP26/GP27**, Relay to **GP10**, OLED to **GP4/GP5**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust either LDR slider down to break a laser beam, check if the alert prints over Bluetooth, then type `'R'` to reset the system.

## Expected Output

Terminal (Bluetooth):
```
Laser Security Grid Online.
ALERT: GRID BREACH - BEAM 1 CUT
SYSTEM RESET: Grid Armed & Secure
```

## Expected Canvas Behavior
* Normal state: OLED reads `LASER GRID HUD` showing LDR sensor readings. Relay is OFF.
* Beam 1 broken (LDR 1 < TRIP_LIMIT): OLED screen fills white, displaying `BREACH!` / `Log: BEAM 1 CUT`. Relay turns ON (Siren ON) and stays latched. Bluetooth sends alerts.

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
This multi-device I2C project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [137 - Pico Intrusion System](../advanced/137-pico-intrusion-system.md)
- [155 - Pico Intrusion System OLED](../advanced/155-pico-intrusion-system-oled.md)
- [166 - Pico Intrusion Datalogger](../advanced/166-pico-intrusion-datalogger.md)
