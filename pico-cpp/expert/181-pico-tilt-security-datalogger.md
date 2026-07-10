# 181 - Pico Tilt Security Datalogger

Build an advanced security vault console that detects physical tilt/vibrations via an MPU-6050, handles keycard checks via an RFID, and logs breaches over Bluetooth.

## Goal
Learn how to pool multiple I2C and SPI sensors (MPU-6050 accelerometer, MFRC522 RFID), drive door servos and alarm sirens, and transmit wireless event warning logs over Bluetooth.

## What You Will Build
An advanced vault security node:
- **MFRC522 RFID Reader**: Checks authorized user keycards.
- **MPU-6050 Accelerometer (GP4, GP5)**: Detects vault tilt or drilling vibration.
- **Servo Motor (GP10)**: Controls the electronic vault door latch.
- **Active Buzzer (GP14)**: Sounds a siren on breach.
- **16x2 I2C LCD (GP4, GP5)**: Displays checks and status logs.
- **HC-05 Bluetooth Module (GP0, GP1)**: Transmits wireless event warnings.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| MPU-6050 IMU | `mpu6050` | Yes | Yes |
| Servo Motor | `servo` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MFRC522 | SDA / SCK / MOSI / MISO / RST | GP17 / GP18 / GP19 / GP16 / GP22 | SPI Bus |
| MPU-6050 | SDA / SCL | GP4 / GP5 | Shared I2C sensor bus |
| Servo Motor | Signal | GP10 | Door latch servo |
| Active Buzzer | VCC (+) | GP14 | Alarm buzzer |
| I2C LCD | SDA / SCL | GP4 / GP5 | Shared I2C display bus |
| HC-05 | TXD / RXD | GP1 / GP0 | UART0 serial lines |
| All Modules | GND | GND | Ground reference |

## Code
```cpp
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <Adafruit_MPU6050.h>
#include <Adafruit_Sensor.h>
#include <LiquidCrystal_I2C.h>

const int RST_PIN   = 22;
const int SS_PIN    = 17;
const int SERVO_PIN = 10;
const int BUZZ_PIN  = 14;

MFRC522 mfrc522(SS_PIN, RST_PIN);
Adafruit_MPU6050 mpu;
Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

const byte authUID[4] = {0xDE, 0x12, 0xA4, 0xC3};

bool alarmLatched = false;
bool vaultUnlocked = false;

void setup() {
  Serial1.begin(9600); // Bluetooth hardware UART

  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  SPI.begin();
  mfrc522.PCD_Init();

  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Locked

  pinMode(BUZZ_PIN, OUTPUT);
  digitalWrite(BUZZ_PIN, LOW);

  lcd.init();
  lcd.backlight();
  
  if (mpu.begin()) {
    lcd.setCursor(0, 0);
    lcd.print("Vault Security");
    lcd.setCursor(0, 1);
    lcd.print("System Active  ");
    Serial1.println("Vault Security System Online.");
  } else {
    lcd.setCursor(0, 0);
    lcd.print("MPU6050 Error!");
    while (1);
  }
  delay(1500);
}

void loop() {
  sensors_event_t a, g, temp;
  mpu.getEvent(&a, &g, &temp);

  // Check tilt thresholds (e.g. absolute acceleration in X or Y > 4.0 m/s^2)
  float absX = a.acceleration.x;
  float absY = a.acceleration.y;
  if (absX < 0) { absX = -absX; }
  if (absY < 0) { absY = -absY; }

  // 1. Detect tilt theft attempts when locked
  if ((absX > 4.0 || absY > 4.0) && !vaultUnlocked && !alarmLatched) {
    alarmLatched = true;
    
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("!! TILT ALERT !!");
    lcd.setCursor(0, 1);
    lcd.print("Vault Tampered! ");
    
    Serial1.println("WARNING: Vault Tamper - Tilt Detected!");
  }

  // 2. Read RFID card scans to disarm / unlock
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    bool match = true;
    if (mfrc522.uid.uidByte[0] != authUID[0]) { match = false; }
    if (mfrc522.uid.uidByte[1] != authUID[1]) { match = false; }
    if (mfrc522.uid.uidByte[2] != authUID[2]) { match = false; }
    if (mfrc522.uid.uidByte[3] != authUID[3]) { match = false; }

    lcd.clear();
    if (match) {
      // Disarm and unlock
      alarmLatched = false;
      vaultUnlocked = true;
      digitalWrite(BUZZ_PIN, LOW);
      lockServo.write(90); // Open door
      
      lcd.setCursor(0, 0);
      lcd.print("Access Granted ");
      lcd.setCursor(0, 1);
      lcd.print("Welcome Back!   ");
      
      Serial1.println("EVENT: Vault Unlocked - Authorized Key");
      tone(BUZZ_PIN, 1000, 300);
      delay(3000);
      
      lockServo.write(0); // Relock
      vaultUnlocked = false;
      showSecuredScreen();
    } else {
      // Denied
      tone(BUZZ_PIN, 200, 600);
      Serial1.println("ALERT: Unauthorized Access Attempt!");
    }
    mfrc522.PICC_HaltA();
  }

  // 3. Siren operation if alarm is active
  if (alarmLatched) {
    digitalWrite(BUZZ_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZ_PIN, LOW);
    delay(100);
  }

  delay(50);
}

void showSecuredScreen() {
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("Vault Security  ");
  lcd.setCursor(0, 1);
  lcd.print("Scan keycard... ");
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MFRC522 RFID Reader**, **MPU-6050 IMU**, **Servo**, **Active Buzzer**, **I2C LCD**, and **HC-05** onto the canvas.
2. Connect MFRC522 SPI pins, MPU-6050/LCD to **GP4/GP5** in parallel, Servo to **GP10**, Buzzer to **GP14**, and HC-05 to **GP0/GP1**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Change the MPU-6050 tilt angles on canvas to trigger the alarm, then scan the authorized RFID tag to disarm the vault.

## Expected Output

Terminal (Bluetooth):
```
Vault Security System Online.
WARNING: Vault Tamper - Tilt Detected!
EVENT: Vault Unlocked - Authorized Key
```

## Expected Canvas Behavior
* Normal state: LCD reads `Scan keycard...`. Servo is at 0°.
* MPU-6050 tilted: LCD reads `!! TILT ALERT !!` / `Vault Tampered!`. Buzzer beeps rapidly. Bluetooth sends alarm.
* Scan Keycard (`DE 12 A4 C3`): LCD reads `Access Granted`, Servo rotates to 90° for 3 seconds, then returns to 0°. Buzzer silences.

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `absX > 4.0 || absY > 4.0` | Accelerometer check that triggers alarms if X or Y axis acceleration deviates significantly from rest states, indicating tilt. |

## Hardware & Safety Concept: Physical Tamper Protection
Safes and security terminals can be physically carried away or broken open by force. Smart vaults use accelerometers (like the MPU-6050) to measure physical orientation. If an intruder attempts to tip, lift, or drill the vault, the sensor detects the acceleration change and triggers alarms, even if the door remains closed.

## Try This! (Challenges)
1. **Calibration Offset**: Add code to calibrate the accelerometer offset on boot, allowing the vault to be mounted on uneven surfaces.
2. **Lockout Latch**: Disable RFID scans for 5 minutes if an unauthorized keycard is scanned 3 times.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers constantly | Tilt threshold set too low | Gravity exerts 9.8 m/s² acceleration. Ensure you isolate the gravity vector or check only for rapid changes (vibration) in code. |

## Mode Notes
This multi-device I2C and SPI project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [87 - Pico RFID Servo](../intermediate/87-pico-rfid-servo.md)
- [108 - Pico MPU6050 Tilt](../intermediate/95-pico-mpu6050-tilt.md)
- [124 - Pico Smart Lock](../advanced/124-pico-smart-lock.md)
- [131 - Pico Anti-Theft](../advanced/131-pico-anti-theft.md)
