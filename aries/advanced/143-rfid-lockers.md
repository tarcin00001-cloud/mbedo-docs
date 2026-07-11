# 143 - RFID Lockers

Create a secure personal locker storage system using an MFRC522 RFID reader to authenticate users, a servo motor to control the physical door latch, a buzzer for audio chime feedback, and an I2C LCD display for user instructions using the VEGA ARIES v3 board.

## Goal
Learn how to maintain persistent toggle state variables in memory, manage dual-direction physical indicators (latch open/closed), execute complex sound sequences, and display system states using an I2C character LCD.

## What You Will Build
A keyless electronic locker. When the simulation starts, the locker remains locked (servo at 0 degrees, LCD showing `Locker: Locked`). 
- **Unlock**: Swiping an authorized card toggles the locker open. The servo rotates to 90 degrees, the buzzer plays two rapid ascending chirps, and the LCD prints `Locker: Unlocked`.
- **Lock**: Swiping the authorized card again toggles the locker closed. The servo rotates back to 0 degrees, the buzzer emits a single long beep, and the LCD returns to `Locker: Locked`.
- **Denied**: An invalid card card triggers a low warning tone and prints `Access Denied` on the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 RFID | SDA (CS) | GP10 | Blue | SPI Chip Select |
| MFRC522 RFID | SCK | GP13 | White | SPI Serial Clock (Shared) |
| MFRC522 RFID | MOSI | GP11 | Yellow | SPI Master Out |
| MFRC522 RFID | MISO | GP12 | Green | SPI Master In |
| MFRC522 RFID | RST | GP9 | Orange | Reset signal |
| MFRC522 RFID | 3.3V | 3.3V | Red | Power supply |
| MFRC522 RFID | GND | GND | Black | Ground reference |
| Servo Motor | Signal | GPIO 13 | Green | PWM control signal (Shared) |
| Servo Motor | VCC | 5V | Red | Power supply (5V) |
| Servo Motor | GND | GND | Black | Ground reference |
| Active Buzzer | VCC (+) | GPIO 14 | Orange | Buzzer control pin |
| Active Buzzer | GND (-) | GND | Grey | Ground reference |
| I2C LCD Display | VCC | 5V | Red | Primary 5V power |
| I2C LCD Display | GND | GND | Black | Ground reference |
| I2C LCD Display | SDA | SDA0 (GP17) | Blue | I2C Data line (Shared) |
| I2C LCD Display | SCL | SCL0 (GP16) | Yellow | I2C Clock line (Shared) |

> **Wiring tip:** Since both the RFID module and the servo motor use shared pins or lines in the simulation, verify power distributions on your breadboard are adequate. Servo movements draw high surge currents that can drop voltage and reset the RFID chip if sharing a weak power source.

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int rstPin = 9;
const int ssPin = 10;
const int servoPin = 13;
const int buzzerPin = 14;

MFRC522 rfid(ssPin, rstPin);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Pre-programmed authorized card bytes
const byte authB0 = 0x3F;
const byte authB1 = 0x5D;
const byte authB2 = 0x8A;
const byte authB3 = 0xB9;

bool isLockerOpen = false;

void setup() {
  Serial.begin(115200);
  delay(1000);

  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(servoPin);
  
  pinMode(buzzerPin, OUTPUT);
  digitalWrite(buzzerPin, LOW);

  // Set default locked state
  lockServo.write(0);

  Wire.begin();
  lcd.init();
  lcd.backlight();
  lcd.print("Locker System");
  delay(1500);
  lcd.clear();
  
  Serial.println("Locker Control Online");
}

void loop() {
  // Show default idle messages on LCD
  lcd.setCursor(0, 0);
  lcd.print("Scan RFID Card  ");
  lcd.setCursor(0, 1);
  if (isLockerOpen) {
    lcd.print("Locker: UNLOCKED");
  } else {
    lcd.print("Locker: LOCKED  ");
  }

  // Poll for RFID swipes
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    byte b0 = rfid.uid.uidByte[0];
    byte b1 = rfid.uid.uidByte[1];
    byte b2 = rfid.uid.uidByte[2];
    byte b3 = rfid.uid.uidByte[3];

    // Check authorization
    if (b0 == authB0 && b1 == authB1 && b2 == authB2 && b3 == authB3) {
      isLockerOpen = !isLockerOpen; // Toggle locker latch state
      
      lcd.clear();
      if (isLockerOpen) {
        lockServo.write(90); // Open door
        Serial.println("Access GRANTED: Unlocking Locker");
        
        // Play double unlock chirp
        digitalWrite(buzzerPin, HIGH); delay(80);
        digitalWrite(buzzerPin, LOW);  delay(80);
        digitalWrite(buzzerPin, HIGH); delay(80);
        digitalWrite(buzzerPin, LOW);
      } else {
        lockServo.write(0); // Close door
        Serial.println("Access GRANTED: Locking Locker");
        
        // Play single lock beep
        digitalWrite(buzzerPin, HIGH); delay(300);
        digitalWrite(buzzerPin, LOW);
      }
    } else {
      Serial.println("Access DENIED: Invalid Card");
      
      lcd.clear();
      lcd.setCursor(0, 0);
      lcd.print("Access Denied!  ");
      lcd.setCursor(0, 1);
      lcd.print("Invalid Token   ");
      
      // Play low error beep
      digitalWrite(buzzerPin, HIGH);
      delay(600);
      digitalWrite(buzzerPin, LOW);
      lcd.clear();
    }
    
    rfid.PICC_HaltA();
  }

  delay(100);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MFRC522 RFID**, **Servo Motor**, **Active Buzzer**, and **I2C LCD** onto the canvas.
2. Wire the RFID module: **3.3V** to **3.3V**, **RST** to **GP9**, **SDA** to **GP10**, **MOSI** to **GP11**, **MISO** to **GP12**, **SCK** to **GP13**, and **GND** to **GND**.
3. Wire the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
4. Wire the Buzzer: **VCC** to **GPIO 14**, and **GND** to **GND**.
5. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
6. Paste the code, select **Interpreted Mode**, and click **Run**.
7. Click the RFID module to swipe cards and watch the lock open and close.

## Expected Output
Serial Monitor:
```
System Initialized.
Locker Control Online
Access GRANTED: Unlocking Locker
Access GRANTED: Locking Locker
Access DENIED: Invalid Card
```

## Expected Canvas Behavior
* The LCD displays `Locker: LOCKED` on startup. The Servo sits at 0 degrees.
* Swiping the authorized card rotates the Servo to 90 degrees and updates the LCD to read `Locker: UNLOCKED`. The buzzer flashes twice.
* Swiping again reverses the servo to 0 degrees and writes `Locker: LOCKED`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `isLockerOpen = !isLockerOpen` | Logical NOT operation toggles the boolean state variable on every valid swipe. |
| `lockServo.write(90)` | Sets the target angle of the servo, rotating the locking arm. |
| `digitalWrite(buzzerPin, HIGH) -> delay` | Generates short voltage pulses on GP14 to drive the alarm buzzer. |
| `rfid.PICC_HaltA()` | Command sent to place the current active transponder into sleep mode to avoid repeating scans. |
| `lcd.print("Locker: LOCKED  ")` | Updates LCD text while overwriting any leftover characters. |

## Hardware & Safety Concept
* **State Latching**: The system maintains the current open/closed state in RAM. This means the system stays in its current state until a new auth token is received, rather than auto-closing like a vault.
* **Inductive Load Protection**: Servos contain DC motors that generate high inductive voltage spikes when starting or stopping. In standalone designs, use a decouple capacitor (100uF - 470uF) across the Servo VCC and GND pins to absorb these spikes.

## Try This! (Challenges)
1. **Intrusion Counter**: Track the number of failed card attempts. If the number of failures exceeds `3`, sound a continuous alarm.
2. **Auto-Relock Mode**: Modify the code so that if the locker remains unlocked for 10 seconds, it sounds warning chirps and locks itself.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servo does not move on card swipe | Power failure or wrong pin | Verify that Servo VCC is connected to 5V and the Signal pin is on GP13. |
| Scanned card is always denied | UID byte mismatch | Print the card UID to serial and copy the bytes exactly into the auth variables in setup. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [142 - Tilt Vault Alarm](142-tilt-vault-alarm.md) (Previous project)
- [144 - RFID Access Logger](144-rfid-access-logger.md) (Next project)
