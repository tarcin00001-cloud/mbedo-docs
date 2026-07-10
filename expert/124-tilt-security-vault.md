# 124 - Tilt Security Vault

Create a dual-protection vault controller that combines an RFID reader (MFRC522) for user access and an accelerometer (MPU-6050) to detect physical tampering. Tilting the vault while locked triggers a security siren and locks down the system.

## Goal
Learn how to read SPI (RFID) and I2C (MPU-6050) sensors concurrently, establish dynamic security states (Locked, Unlocked, Alarm), and apply strict variable naming constraints (`Ac_x`, `Ac_z`) required for interpreter shims.

## What You Will Build
An anti-tamper security vault:
- **RFID Verification**: Swiping the correct keycard unlocks the vault door (servo swings to 90 degrees).
- **Tilt Detection**: If the vault is tilted (`Ac_x` or `Ac_z` values fluctuate beyond safety boundaries) *while locked*, the system enters alarm lockdown.
- **Lockdown Action**: Sounds a continuous buzzer alarm, flashes warning indicators, and locks down the interface until reset.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MFRC522 RFID | `mfrc522` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| RFID Reader | 3.3V | 3.3V | Power supply |
| RFID Reader | RST | D9 | Reset control |
| RFID Reader | SDA (SS) | D10 | SPI Slave Select |
| RFID Reader | MOSI | D11 | SPI MOSI |
| RFID Reader | MISO | D12 | SPI MISO |
| RFID Reader | SCK | D13 | SPI Clock |
| RFID Reader | GND | GND | Ground reference |
| MPU-6050 IMU | VCC | 5V | Power supply |
| MPU-6050 IMU | SDA | A4 | I2C Data bus |
| MPU-6050 IMU | SCL | A5 | I2C Clock bus (shared) |
| MPU-6050 IMU | GND | GND | Ground reference |
| Servo Motor | Signal | D6 | PWM servo signal |
| Active Buzzer | VCC | D8 | Alarm feedback pin |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus (shared) |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus (shared) |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Servo.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN    = 9;
const int SS_PIN     = 10;
const int SERVO_PIN  = 6;
const int BUZZER_PIN = 8;

// MPU-6050 I2C Address
const int MPU_ADDR = 0x68;

MFRC522 rfid(SS_PIN, RST_PIN);
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Master keycard UID bytes (e.g. 0x3F 0x5D 0x8A 0xB9)
const byte AUTH_B0 = 0x3F;
const byte AUTH_B1 = 0x5D;
const byte AUTH_B2 = 0x8A;
const byte AUTH_B3 = 0xB9;

bool isUnlocked = false;
bool isTampered = false;

// Naming convention variables required for interpreter shims
int16_t Ac_x, Ac_y, Ac_z;
int16_t Gy_x, Gy_y, Gy_z;
int16_t Tmp;

void setup() {
  Serial.begin(9600);
  SPI.begin();
  rfid.PCD_Init();
  lockServo.attach(SERVO_PIN);
  pinMode(BUZZER_PIN, OUTPUT);

  lockServo.write(0); // Lock door
  digitalWrite(BUZZER_PIN, LOW);

  // Initialize MPU-6050
  Wire.begin();
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x6B); // PWR_MGMT_1 register
  Wire.write(0);    // set to zero (wakes up the MPU-6050)
  Wire.endTransmission(true);

  lcd.init();
  lcd.backlight();
  lcd.print("Vault Security");
  delay(1500);
  lcd.clear();

  Serial.println("Vault Security System Online");
}

void loop() {
  // Read MPU-6050 Accelerometer
  Wire.beginTransmission(MPU_ADDR);
  Wire.write(0x3B); // starting register for Accelerometer data
  Wire.endTransmission(false);
  Wire.requestFrom(MPU_ADDR, 14, true);

  // Read raw acceleration values (uses required variable names)
  Ac_x = Wire.read() << 8 | Wire.read();
  Ac_y = Wire.read() << 8 | Wire.read();
  Ac_z = Wire.read() << 8 | Wire.read();
  Tmp  = Wire.read() << 8 | Wire.read();
  Gy_x = Wire.read() << 8 | Wire.read();
  Gy_y = Wire.read() << 8 | Wire.read();
  Gy_z = Wire.read() << 8 | Wire.read();

  // Tamper detection: trigger if tilted significantly (flat baseline Ac_x ~ 0, range +/-16384)
  if (!isUnlocked && !isTampered) {
    if (abs(Ac_x) > 4000 || abs(Ac_y) > 4000) {
      isTampered = true;
      Serial.println("ALERT: Vault Tampered / Tilted!");
    }
  }

  // State loop processing
  if (isTampered) {
    digitalWrite(BUZZER_PIN, HIGH); // Alarm sounds
    lockServo.write(0);             // Force locked state
    
    lcd.setCursor(0, 0);
    lcd.print("VAULT TAMPERED! ");
    lcd.setCursor(0, 1);
    lcd.print("ALARM LOCKED    ");
  } else if (isUnlocked) {
    digitalWrite(BUZZER_PIN, LOW);
    lockServo.write(90); // Swings open
    
    lcd.setCursor(0, 0);
    lcd.print("Vault Open      ");
    lcd.setCursor(0, 1);
    lcd.print("Access Granted  ");
  } else {
    // Standard locked monitoring
    digitalWrite(BUZZER_PIN, LOW);
    lockServo.write(0);

    lcd.setCursor(0, 0);
    lcd.print("Scan RFID Card  ");
    lcd.setCursor(0, 1);
    lcd.print("Vault Guarded   ");

    if (rfid.PICC_IsNewCardPresent()) {
      if (rfid.PICC_ReadCardSerial()) {
        byte b0 = rfid.uid.uidByte[0];
        byte b1 = rfid.uid.uidByte[1];
        byte b2 = rfid.uid.uidByte[2];
        byte b3 = rfid.uid.uidByte[3];

        if (b0 == AUTH_B0 && b1 == AUTH_B1 && b2 == AUTH_B2 && b3 == AUTH_B3) {
          isUnlocked = true;
          Serial.println("Authorized Access Granted.");
          lcd.clear();
        } else {
          Serial.println("Unauthorized Card!");
          
          digitalWrite(BUZZER_PIN, HIGH);
          delay(500);
          digitalWrite(BUZZER_PIN, LOW);

          lcd.setCursor(0, 0);
          lcd.print("Access Denied   ");
          delay(1500);
          lcd.clear();
        }
        rfid.PICC_HaltA();
      }
    }
  }

  // Serial command reset
  if (Serial.available()) {
    char cmd = Serial.read();
    if (cmd == 'r' || cmd == 'R') {
      isTampered = false;
      isUnlocked = false;
      Serial.println("System Reset Successful.");
      lcd.clear();
    }
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MFRC522 RFID**, **MPU-6050 IMU**, **Servo Motor**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect RFID: **3.3V** to **3.3V** (or **3V3** pin), **RST** to **D9**, **SDA** (SS) to **D10**, **MOSI** to **D11**, **MISO** to **D12**, **SCK** to **D13**, **GND** to **GND**.
3. Connect MPU-6050: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
4. Connect Servo: **Signal** to **D6**, **VCC** to **5V**, **GND** to **GND**.
5. Connect Buzzer: **VCC** to **D8**, **GND** to **GND**.
6. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
7. Paste code, select the interpreted mode, and click **Run**.
8. Scan card to unlock, or modify MPU-6050 tilt values to trigger the tamper alarm.
9. Enter `r` in the serial monitor to reset the vault states.

## Expected Output

Terminal:
```
Vault Security System Online
ALERT: Vault Tampered / Tilted!
System Reset Successful.
```

LCD Display:
```
VAULT TAMPERED! 
ALARM LOCKED    
```

## Expected Canvas Behavior
| Card Scan | MPU-6050 Tilt (Ac_x) | Tamper State | Servo Angle | Buzzer (D8) | LCD Line 1 |
| --- | --- | --- | --- | --- | --- |
| None | 0 | FALSE | 0 deg | LOW | `Scan RFID Card` |
| Valid card | 0 | FALSE | 90 deg | LOW | `Vault Open` |
| None | 5000 (Tilted) | TRUE | 0 deg | HIGH | `VAULT TAMPERED!` |
| Valid card | 5000 (Tilted) | TRUE | 0 deg | HIGH | `VAULT TAMPERED!` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `Wire.write(0x6B) -> Wire.write(0)` | Wakes up the MPU-6050 accelerometer from deep sleep mode to begin active measurements. |
| `abs(Ac_x) > 4000` | Checks if horizontal axis tilt forces accelerate beyond the ~0.24g safety threshold. |
| `lockServo.write(90)` | Rotates the locking arm to open the deadbolt lock mechanism. |

## Hardware & Safety Concept: Anti-Tamper Accelerometers
Physical security vaults use high-frequency accelerometers to track structural displacement or micro-vibrations generated by drills, hammers, or tilting. Using I2C modules like the MPU-6050, the controller can sample tilt states along three separate axes, locking down critical actuators like deadbolt servos before an intruder manages to bypass secondary locking boundaries.

## Try This! (Challenges)
1. **Shaking Trigger**: Create a rolling delta system. If the acceleration changes more than 8000 units within two cycles (detecting structural hitting/shaking), trigger the alarm immediately.
2. **Door Open Timeout**: Monitor the servo state. If the vault is unlocked for more than 10 seconds, sound a warning chirp to alert the user to close the vault door.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Accelerometer always reads 0 | Wakeup command omitted | Confirm that register `0x6B` is set to `0` in `setup()` to wake the sensor. |
| Tamper alarm trips on start | Incorrect threshold baseline | Make sure the baseline checks allow for standard gravity calibration offsets (Ac_z sits near 16384 when flat). |

## Mode Notes
This multi-device anti-tamper structure runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [70 - RFID Servo Unlock](../intermediate/70-rfid-servo-unlock.md)
- [105 - Tilt Alarm](../advanced/105-tilt-alarm.md)
- [111 - Locker System](../advanced/111-locker-system.md)
