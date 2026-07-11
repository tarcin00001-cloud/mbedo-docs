# 142 - Tilt Vault Alarm

Implement a security vault system that requires RFID verification for opening and utilizes a 3-axis accelerometer to detect physical tilting or tampering, displaying alerts on an LCD and driving a lock servo using the VEGA ARIES v3 board.

## Goal
Learn how to manage multiple serial interfaces (SPI for RFID, I2C for accelerometer and LCD) in a single application, handle device state synchronization, read accelerometer registers directly, and control physical deadbolt actuators.

## What You Will Build
An anti-tamper security vault:
- **Lock Deadbolt**: Controlled by a servo motor. Starts at 0 degrees (Locked).
- **RFID Access**: Swiping an authorized RFID card rotates the servo to 90 degrees (Unlocked) and updates the LCD display.
- **Tilt Detection**: If the vault is tilted beyond limits while locked, the system enters lockdown state, flashes warnings on the LCD, and refuses access until manually reset.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| MPU-6050 IMU Sensor | `mpu6050` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
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
| MPU-6050 IMU | VCC | 3V3 | Orange | Power supply (3.3V) |
| MPU-6050 IMU | GND | GND | Grey | Ground reference |
| MPU-6050 IMU | SDA | SDA0 (GP17) | Blue | I2C Data bus (Shared) |
| MPU-6050 IMU | SCL | SCL0 (GP16) | Yellow | I2C Clock bus (Shared) |
| Servo Motor | Signal | GPIO 13 | Green | PWM control signal (Shared) |
| Servo Motor | VCC | 5V | Red | Primary 5V power |
| Servo Motor | GND | GND | Black | Ground reference |
| I2C LCD Display | VCC | 5V | Red | Primary 5V power |
| I2C LCD Display | GND | GND | Black | Ground reference |
| I2C LCD Display | SDA | SDA0 (GP17) | Blue | I2C Data line (Shared) |
| I2C LCD Display | SCL | SCL0 (GP16) | Yellow | I2C Clock line (Shared) |

> **Wiring tip:** The servo signal pin connects to GPIO 13. Since SPI SCK also maps to GP13, this highlights how pins are allocated for multiple actuators. In real hardware implementations, ensure you map the servo to an independent PWM pin if SPI is active.

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
const int mpuAddr = 0x68;

MFRC522 rfid(ssPin, rstPin);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Master card authorization bytes (e.g. 0x3F, 0x5D, 0x8A, 0xB9)
const byte authB0 = 0x3F;
const byte authB1 = 0x5D;
const byte authB2 = 0x8A;
const byte authB3 = 0xB9;

bool isUnlocked = false;
bool isTampered = false;

// Required global variables for MPU-6050 registers
int16_t Ac_x, Ac_y, Ac_z;
int16_t Gy_x, Gy_y, Gy_z;
int16_t Tmp;

void setup() {
  Serial.begin(115200);
  delay(1000);

  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(servoPin);
  
  // Set servo to Locked position initially
  lockServo.write(0);

  // Initialize MPU-6050
  Wire.begin();
  Wire.beginTransmission(mpuAddr);
  Wire.write(0x6B); // Wake up register
  Wire.write(0);    // Clear sleep bit
  Wire.endTransmission(true);

  lcd.init();
  lcd.backlight();
  lcd.print("Vault Security");
  delay(1500);
  lcd.clear();
  
  Serial.println("Vault Security System Online");
}

void loop() {
  // Read Accelerometer values from MPU-6050
  Wire.beginTransmission(mpuAddr);
  Wire.write(0x3B); // Starting register for accelerometer
  Wire.endTransmission(false);
  Wire.requestFrom(mpuAddr, 14, true);

  Ac_x = Wire.read() << 8 | Wire.read();
  Ac_y = Wire.read() << 8 | Wire.read();
  Ac_z = Wire.read() << 8 | Wire.read();
  Tmp  = Wire.read() << 8 | Wire.read();
  Gy_x = Wire.read() << 8 | Wire.read();
  Gy_y = Wire.read() << 8 | Wire.read();
  Gy_z = Wire.read() << 8 | Wire.read();

  // Tamper detection: check if vault is tilted while locked
  if (!isUnlocked && !isTampered) {
    if (abs(Ac_x) > 4000 || abs(Ac_y) > 4000) {
      isTampered = true;
      Serial.println("ALERT: Vault Tampered / Tilted!");
    }
  }

  // Handle security states
  if (isTampered) {
    lockServo.write(0); // Force locked deadbolt
    
    lcd.setCursor(0, 0);
    lcd.print("VAULT TAMPERED! ");
    lcd.setCursor(0, 1);
    lcd.print("ALARM LOCKED    ");
  } else if (isUnlocked) {
    lockServo.write(90); // Rotate servo to release lock
    
    lcd.setCursor(0, 0);
    lcd.print("Vault Open      ");
    lcd.setCursor(0, 1);
    lcd.print("Access Granted  ");
  } else {
    lockServo.write(0); // Keep locked
    
    lcd.setCursor(0, 0);
    lcd.print("Scan RFID Card  ");
    lcd.setCursor(0, 1);
    lcd.print("Vault Guarded   ");

    // Scan for card attempts
    if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
      byte b0 = rfid.uid.uidByte[0];
      byte b1 = rfid.uid.uidByte[1];
      byte b2 = rfid.uid.uidByte[2];
      byte b3 = rfid.uid.uidByte[3];

      if (b0 == authB0 && b1 == authB1 && b2 == authB2 && b3 == authB3) {
        isUnlocked = true;
        Serial.println("Authorized Access Granted.");
        lcd.clear();
      } else {
        Serial.println("Unauthorized Card Attempt!");
        
        lcd.setCursor(0, 0);
        lcd.print("Access Denied   ");
        delay(1500);
        lcd.clear();
      }
      rfid.PICC_HaltA();
    }
  }

  // Reset console input parser
  if (Serial.available()) {
    char cmd = Serial.read();
    if (cmd == 'r' || cmd == 'R') {
      isTampered = false;
      isUnlocked = false;
      Serial.println("Vault Alarm Reset Successful.");
      lcd.clear();
    }
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **MFRC522 RFID**, **MPU-6050 IMU**, **Servo Motor**, and **I2C LCD** onto the canvas.
2. Wire the RFID module: **3.3V** to **3.3V**, **RST** to **GP9**, **SDA** to **GP10**, **MOSI** to **GP11**, **MISO** to **GP12**, **SCK** to **GP13**, and **GND** to **GND**.
3. Wire the MPU-6050: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Wire the Servo: **Signal** to **GPIO 13**, **VCC** to **5V**, and **GND** to **GND**.
5. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
6. Paste the code, select **Interpreted Mode**, and click **Run**.
7. Click the RFID module to swipe cards, or click the MPU-6050 to change tilt parameters to verify responses.

## Expected Output
Serial Monitor:
```
System Initialized.
Vault Security System Online
ALERT: Vault Tampered / Tilted!
Vault Alarm Reset Successful.
Authorized Access Granted.
```

## Expected Canvas Behavior
* On startup, the LCD reads `Scan RFID Card`. The Servo locks at 0 degrees.
* When a valid card matches the authorized bytes, the Servo turns to 90 degrees.
* If the MPU-6050 is tilted, the LCD flashes `VAULT TAMPERED!`, the Servo snaps to 0, and the status changes to lockdown.
* Sending `r` resets the alarm and locks the vault back to standard monitoring.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Wire.write(0x6B) -> Wire.write(0)` | Disables the sleep register bit to activate raw sensor reads. |
| `Ac_x = Wire.read() << 8 \| Wire.read()` | Reconstructs 16-bit signed integer values from two separate 8-bit registers. |
| `abs(Ac_x) > 4000` | Checks if tilt deviations exceed the safety threshold. |
| `rfid.PICC_IsNewCardPresent()` | Polls for RFID antennas entering the RF induction field. |
| `lockServo.write(90)` | Directs the servo to rotate, moving the physical locking mechanism. |

## Hardware & Safety Concept
* **Lockdown Mechanics**: Once a security breach is detected, a software latch (`isTampered`) prevents standard access until a manual reset command is typed into the Serial console. This prevents intruders from simply returning the vault to a level position to bypass alarms.
* **Bus Contention**: The I2C bus utilizes hardware arbitration to manage lines. Both the LCD and MPU-6050 function smoothly as long as their unique hex addresses do not overlap.

## Try This! (Challenges)
1. **Auto-Relock Timer**: Automatically relock the vault door (servo back to 0 degrees and set `isUnlocked` to false) 5 seconds after a successful card unlock.
2. **Double Card Auth**: Require two different valid card swipes within 10 seconds of each other to successfully unlock the vault door.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD does not print MPU tilt | Sensor in deep sleep | Verify the wake command in setup has been executed properly on register `0x6B`. |
| Card scan does nothing | SPI conflict | Double check that CS is wired to GP10 and RST goes to GP9. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [141 - Sounding Sentry Guard](141-sounding-sentry-guard.md) (Previous project)
- [143 - RFID Lockers](143-rfid-lockers.md) (Next project)
