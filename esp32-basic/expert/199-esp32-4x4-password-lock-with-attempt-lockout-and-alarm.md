# 199 - ESP32 4x4 Password Lock with Attempt Lockout & Alarm

Build a commercial-grade keypad lock on the ESP32 that unlocks a servo deadbolt on a correct 4-digit PIN match, tracks incorrect entry attempts, and enters a 60-second security lockout state with a buzzer alarm siren if three consecutive incorrect PINs are entered.

## Goal
Learn how to implement attempt counter states, run non-blocking lockout countdown timers, ignore inputs during safety lockouts, and trigger sirens.

## What You Will Build
A 4x4 matrix keypad is connected to 8 GPIO pins. A lock servo is on GPIO 27, a buzzer on GPIO 4, and a 16x2 LCD on I2C. The user enters `5555#` to unlock the deadbolt. If three incorrect codes are entered, the system locks out for 60 seconds, sounding a siren for the first 10 seconds and displaying a countdown timer on the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 10 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Rows | Pins 1–4 | GPIO12, 13, 14, 15 | Red/Orange/Yellow/Green | Keypad row lines |
| Keypad Cols | Pins 5–8 | GPIO16, 17, 25, 26 | Blue/Purple/White/Gray | Keypad col lines |
| Servo Motor | Signal | GPIO27 | Brown | PWM lock deadbolt control |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Alarm / status horn |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the I2C bus. Power the deadbolt servo from the 5V Vin rail to prevent voltage drops.

## Code
```cpp
// 4x4 Password Lock with Attempt Lockout & Alarm
#include <Keypad.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int SERVO_PIN = 27;
const int BUZZER_PIN = 4;

LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo lockServo;

// Keypad Configuration
const byte ROWS = 4;
const byte COLS = 4;
char keys[ROWS][COLS] = {
  {'1','2','3','A'},
  {'4','5','6','B'},
  {'7','8','9','C'},
  {'*','0','#','D'}
};
byte rowPins[ROWS] = {12, 13, 14, 15};
byte colPins[COLS] = {16, 17, 25, 26};
Keypad keypad = Keypad(makeKeymap(keys), rowPins, colPins, ROWS, COLS);

// Secret PIN definition
const String SECRET_PIN = "5555";

// Security variables
String inputBuffer = "";
int failedAttempts = 0;
const int MAX_ATTEMPTS = 3;

// Lockout timer parameters
bool isLockedOut = false;
unsigned long lockoutStartTime = 0;
const unsigned long LOCKOUT_DURATION_MS = 60000; // 60-second lockout

void playChime(bool success) {
  if (success) {
    digitalWrite(BUZZER_PIN, HIGH); delay(80);
    digitalWrite(BUZZER_PIN, LOW);  delay(50);
    digitalWrite(BUZZER_PIN, HIGH); delay(150);
    digitalWrite(BUZZER_PIN, LOW);
  } else {
    digitalWrite(BUZZER_PIN, HIGH); delay(400);
    digitalWrite(BUZZER_PIN, LOW);
  }
}

void triggerLockout() {
  Serial.println("!!! SECURITY ALERT: LOCKOUT ACTIVATED !!!");
  isLockedOut = true;
  lockoutStartTime = millis();
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("SYSTEM LOCKED!  ");
  
  // Sound alarm siren for the first 10 seconds
  for (int i = 0; i < 20; i++) { // 20 * 500ms = 10s
    digitalWrite(BUZZER_PIN, HIGH); delay(250);
    digitalWrite(BUZZER_PIN, LOW);  delay(250);
  }
}

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  lcd.init();
  lcd.backlight();
  
  Serial.println("Password Security System online.");
}

void loop() {
  unsigned long now = millis();
  
  // 1. Evaluate Lockout state
  if (isLockedOut) {
    unsigned long timeElapsed = now - lockoutStartTime;
    
    if (timeElapsed >= LOCKOUT_DURATION_MS) {
      // Lockout expired: reset variables
      isLockedOut = false;
      failedAttempts = 0;
      inputBuffer = "";
      Serial.println("Lockout expired. System restored.");
      lcd.clear();
    } else {
      // Display remaining countdown time
      int secondsRemaining = (LOCKOUT_DURATION_MS - timeElapsed) / 1000;
      
      lcd.setCursor(0, 0);
      lcd.print("LOCKOUT ACTIVE! ");
      lcd.setCursor(0, 1);
      lcd.print("Retry in: ");
      lcd.print(secondsRemaining);
      lcd.print("s   ");
      
      delay(200);
      return; // Ignore keypad inputs during lockout
    }
  }
  
  // 2. Read Keypad Input
  char key = keypad.getKey();
  
  if (key) {
    if (key == '#') {
      if (inputBuffer == SECRET_PIN) {
        Serial.println("PIN Match. Unlocking deadbolt.");
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("ACCESS GRANTED  ");
        lcd.setCursor(0, 1);
        lcd.print("Deadbolt Open   ");
        
        playChime(true);
        lockServo.write(90); // Unlock
        delay(3000);         // Keep open for 3 seconds
        lockServo.write(0);  // Relock
        
        failedAttempts = 0;  // Reset attempts
        lcd.clear();
      } else {
        failedAttempts++;
        Serial.print("PIN incorrect! Failed attempts: "); Serial.println(failedAttempts);
        
        lcd.clear();
        lcd.setCursor(0, 0);
        lcd.print("ACCESS DENIED!  ");
        lcd.setCursor(0, 1);
        lcd.print("Attempts: "); lcd.print(failedAttempts);
        
        playChime(false);
        delay(1500);
        lcd.clear();
        
        // Trigger lockout if limit reached
        if (failedAttempts >= MAX_ATTEMPTS) {
          triggerLockout();
        }
      }
      inputBuffer = ""; // Clear buffer
    } 
    else if (key == '*') {
      inputBuffer = ""; // Reset buffer
    } 
    else {
      // Append number keys (limit to 4 digits)
      if (inputBuffer.length() < 4) {
        inputBuffer += key;
        // Sound a quick beep on press
        digitalWrite(BUZZER_PIN, HIGH); delay(30); digitalWrite(BUZZER_PIN, LOW);
      }
    }
  }
  
  // 3. Update normal operation HUD
  lcd.setCursor(0, 0);
  lcd.print("Enter Password: ");
  lcd.setCursor(0, 1);
  for (int i = 0; i < inputBuffer.length(); i++) lcd.print("*");
  for (int i = inputBuffer.length(); i < 4; i++) lcd.print(" ");
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Keypad**, **Servo**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire Keypad rows/cols, Servo to **GPIO27**, Buzzer to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Type `5555#` on the keypad widget. Watch the servo deadbolt open for 3 seconds.
5. Type three incorrect codes (e.g. `1111#`, `2222#`, `3333#`). Watch the system enter lockout, sound the buzzer alarm, and start the LCD countdown.

## Expected Output
Serial Monitor:
```
Password Security System online.
PIN Match. Unlocking deadbolt.
PIN incorrect! Failed attempts: 1
PIN incorrect! Failed attempts: 2
PIN incorrect! Failed attempts: 3
!!! SECURITY ALERT: LOCKOUT ACTIVATED !!!
Lockout expired. System restored.
```

LCD Display (idle):
```
Enter Password:
****
```

LCD Display (lockout):
```
LOCKOUT ACTIVE!
Retry in: 48s
```

## Expected Canvas Behavior
* At boot, the LCD displays `Enter Password:`. The servo is at 0°.
* Typing `5555#` on the keypad widget sweeps the servo to 90° and plays a chime.
* Entering three incorrect codes causes the buzzer widget to flash green (alarm) for 10 seconds, and the LCD counts down from 60 seconds.
* Keypad clicks are ignored during the 60-second lockout.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `inputBuffer == SECRET_PIN` | Verifies the typed code against the authorized PIN definition. |
| `failedAttempts++` | Increments the incorrect attempt counter. |
| `triggerLockout()` | Triggers the safety lockout state, starting the countdown timer. |
| `return;` | Exits the loop early during lockout, ignoring keypad inputs. |

## Hardware & Safety Concept: Keypad Brute-Force Attacks and Lockout Loops
Keypads are vulnerable to **brute-force attacks** (where an intruder tries random codes until one matches). To protect the system, the controller must limit the number of consecutive incorrect entries. If the limit (e.g. 3) is exceeded, the system triggers a **lockout state**, freezing the keypad and sounding a siren. This prevents automated attacks by enforcing a delay between attempts.

## Try This! (Challenges)
1. **Interactive PIN change**: Implement a menu sequence (e.g. `*7777#`) allowing users to change the password PIN code.
2. **SD Card Intruder Logger**: Log all failed entry attempts and lockout event timestamps to an SD card (Project 144).
3. **Lockout time scaling**: Double the lockout duration (e.g. 120 seconds) on subsequent failures.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad buttons don't register | Keypad rows/columns miswired | Check keypad pinout connections and verify GPIO assignments 12–15 and 16, 17, 25, 26 |
| Buzzer does not sound | Buzzer pin misconfigured | Confirm active buzzer is wired to GPIO 4 |
| Servo does not move | Current draw overload | Power the servo from the 5V Vin rail, not the 3.3V rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [114 - ESP32 Keypad 4x4 Password Lock](../intermediate/114-esp32-keypad-4x4-password-lock.md)
- [158 - ESP32 Keypad RFID Multi-factor Lock](158-esp32-keypad-rfid-multi-factor-lock.md)
- [168 - ESP32 Bluetooth Keypad Entry Lock](168-esp32-bluetooth-keypad-entry-lock.md)
