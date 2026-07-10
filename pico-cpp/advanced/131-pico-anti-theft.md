# 131 - Pico Anti-Theft

Build an automotive anti-theft ignition lock that disables a starter relay on vibration tampering and requires a keycard validation swipe to reset.

## Goal
Learn how to implement high-reliability security latching rules combining SPI RFID readers, digital vibration sensors, and starter interlock relays.

## What You Will Build
An automotive immobilizer control module:
- **MFRC522 RFID Reader**: Scans the driver's keycard.
- **Relay Module (GP10)**: Connects to the vehicle's ignition starter circuit.
- **Vibration Sensor (GP28)**: Detects door tampering or window break shock events.
- **Active Buzzer (GP14)**: Sounds a siren on unauthorized vehicle entry.
- **Red LED (GP15)**: Warns of tampered system lock states.
- **16x2 I2C LCD (GP4, GP5)**: Displays status readouts (e.g. "IGNITION ARMED" or "ALARM! TAMPERED").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Vibration Sensor Module | `button` | Yes (represented by switch button) | Yes (SW-420 sensor) |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| LED (Red) | `led` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| Vibration Sensor | OUT | GP28 | Shock detection |
| Relay Module | IN | GP10 | Ignition cut-off line |
| Active Buzzer | VCC (+) | GP14 | Siren pin |
| Red LED | Anode | GP15 | Security flash |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C screen bus |
| All Modules | GND | GND | Ground return |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int VIB_PIN    = 28;
const int RELAY_PIN = 10;
const int BUZZ_PIN  = 14;
const int RED_LED   = 15;

MFRC522 mfrc522(SS_PIN, RST_PIN);
LiquidCrystal_I2C lcd(0x27, 16, 2);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

bool systemArmed = true;
bool alarmActive = false;

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  pinMode(VIB_PIN, INPUT);
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZ_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);

  // Lock ignition on startup (relay OPEN)
  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZ_PIN, LOW);
  digitalWrite(RED_LED, HIGH); // Armed indicator ON

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Immobilizer Sys");
  lcd.setCursor(0, 1);
  lcd.print("State: ARMED    ");
}

void loop() {
  int tamper = digitalRead(VIB_PIN); // Active-LOW spring check

  // 1. Detect tampering when armed
  if (systemArmed && tamper == LOW && !alarmActive) {
    alarmActive = true;
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("!! ALARM TRIPPED !!");
    lcd.setCursor(0, 1);
    lcd.print("Scan Keycard... ");
  }

  // 2. Read RFID card scans to disarm / silence system
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    if (match) {
      // Disarm and unlock ignition
      systemArmed = false;
      alarmActive = false;
      digitalWrite(RELAY_PIN, HIGH); // Allow starter ignition
      digitalWrite(BUZZ_PIN, LOW);
      digitalWrite(RED_LED, LOW);   // Armed indicator OFF
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Key Verified");
      lcd.setCursor(0, 1);
      lcd.print("Ignition: ENABLE");
      tone(BUZZ_PIN, 1000, 300);
      delay(3000);
    } else {
      // Mismatch card alert
      tone(BUZZ_PIN, 200, 600);
    }
    mfrc522.PICC_HaltA();
  }

  // 3. Siren operation
  if (alarmActive) {
    digitalWrite(BUZZ_PIN, HIGH);
    digitalWrite(RED_LED, HIGH);
    delay(100);
    digitalWrite(BUZZ_PIN, LOW);
    digitalWrite(RED_LED, LOW);
    delay(100);
  } else {
    // Normal blinking arm indicator
    if (systemArmed) {
      digitalWrite(RED_LED, HIGH);
      delay(200);
      digitalWrite(RED_LED, LOW);
      delay(200);
    }
  }

  delay(20);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **Vibration Sensor** (switch button), **Relay**, **Active Buzzer**, **Red LED**, and **I2C LCD** onto the canvas.
2. Connect MFRC522 to SPI pins, Vibration to **GP28**, Relay to **GP10**, Buzzer to **GP14**, LED to **GP15**, and LCD to **GP4/GP5**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Simulate safe tampering by closing the vibration switch, then scan the authorized RFID tag to disarm the alarm and enable the starter relay.

## Expected Output

Terminal:
```
Simulation active. Vehicle immobilizer node online.
```

## Expected Canvas Behavior
* Startup: LCD reads `State: ARMED`, Red LED flashes slowly. Relay is OFF (Starter cut).
* Vibration triggered: LCD flashes `!! ALARM TRIPPED !!`, Buzzer and LED flash rapidly in sync.
* Scan Keycard (`DE 12 A4 C3`): LCD reads `Ignition: ENABLE`, Relay closes (GP10 HIGH), system falls silent.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Closes the starter interlock relay, enabling the ignition circuit. |

## Hardware & Safety Concept: Automotive Interlock Circuits
Automotive immobilizer systems cut power to critical engine components (such as the fuel pump or starter solenoid) using interlock relays. To prevent vehicle theft, these relays are wired in a **Normally Open (NO)** configuration, meaning power is cut off if the security control module loses power or is removed.

## Try This! (Challenges)
1. **Auto Re-Arm**: Automatically re-arm the system (relay OFF, blinking arm LED) if the vehicle is turned off or if no key is scanned within 15 seconds.
2. **Alert Logger**: Send warning messages over a serial Bluetooth bridge when tampering is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Tamper alarm triggers under normal engine vibration | Sensitivity dial set too low | When using spring vibration sensors, adjust the physical trimmer potentiometer to prevent normal vehicle vibrations from triggering the alarm. |

## Mode Notes
This multi-device I2C, SPI, and multiplexed project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [87 - Pico RFID Servo](../intermediate/87-pico-rfid-servo.md)
- [106 - Pico Shake Alarm](../intermediate/106-pico-shake-alarm.md)
- [124 - Pico Smart Lock](124-pico-smart-lock.md)
