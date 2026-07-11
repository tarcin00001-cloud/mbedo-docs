# 168 - ESP32 Bluetooth Keypad Entry Lock

Build a hybrid smart lock that can be unlocked using a physical 4x4 keypad PIN code or remotely using serial commands sent via a Bluetooth link, using a servo motor deadbolt and status display.

## Goal
Learn how to manage multiple input sources (local matrix keypad and remote Bluetooth serial), parse command tokens, and build a unified unlocking logic.

## What You Will Build
A 4x4 matrix keypad is connected to 8 GPIO pins. A Bluetooth module (HC-05 or built-in Bluetooth Classic Serial) is connected to UART2 (RX2: 16, TX2: 17). A servo is on GPIO 13, a buzzer on GPIO 4, and a 16x2 LCD on I2C. The deadbolt unlocks when the code "4321#" is typed on the keypad or the command `UNLOCK` is received over Bluetooth, relocking automatically after 4 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| 4x4 Matrix Keypad | `keypad` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes | Yes |
| Servo Motor (SG90) | `servo` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Keypad Rows | Pins 1–4 | GPIO12, 13, 14, 15 | Red/Orange/Yellow/Green | Keypad row lines |
| Keypad Cols | Pins 5–8 | GPIO16, 17, 25, 26 | Blue/Purple/White/Gray | Keypad col lines |
| HC-05 Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | Bluetooth serial |
| Servo Motor | Signal | GPIO13 | Orange | PWM lock actuator |
| Active Buzzer | VCC (+) | GPIO4 | Blue | Success/Error chime |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Share the RX2 (GPIO 16) and TX2 (GPIO 17) lines with the HC-05. The LCD connects to the I2C bus (GPIO 21/22). Power the servo from the 5V Vin rail.

## Code
```cpp
// Bluetooth Keypad Entry Lock (Keypad + BT + Servo + LCD)
#include <Keypad.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <ESP32Servo.h>

const int SERVO_PIN = 13;
const int BUZZER_PIN = 4;

#define RX2_PIN 16
#define TX2_PIN 17

Servo lockServo;
LiquidCrystal_I2C lcd(0x27, 16, 2);

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

// Authorized keys definitions
const String SECRET_PIN = "4321";
const String BT_UNLOCK_CMD = "UNLOCK";

String keypadInput = "";
String bluetoothInput = "";

void playSuccessChime() {
  digitalWrite(BUZZER_PIN, HIGH); delay(100);
  digitalWrite(BUZZER_PIN, LOW);  delay(50);
  digitalWrite(BUZZER_PIN, HIGH); delay(100);
  digitalWrite(BUZZER_PIN, LOW);
}

void playErrorChime() {
  digitalWrite(BUZZER_PIN, HIGH); delay(400);
  digitalWrite(BUZZER_PIN, LOW);
}

void unlockDoor(const char * source) {
  Serial.print("Door Unlocked by: "); Serial.println(source);
  
  lcd.clear();
  lcd.setCursor(0, 0);
  lcd.print("ACCESS GRANTED  ");
  lcd.setCursor(0, 1);
  lcd.print("By: "); lcd.print(source);
  
  playSuccessChime();
  lockServo.write(90); // Open deadbolt
  
  delay(4000); // Keep open for 4 seconds
  
  lockServo.write(0); // Relock deadbolt
  Serial.println("Door Auto-Relocked.");
  lcd.clear();
}

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN); // Bluetooth module Serial link
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  lockServo.attach(SERVO_PIN);
  lockServo.write(0); // Start locked
  
  lcd.init();
  lcd.backlight();
  
  Serial.println("Hybrid Smart Lock online. Ready for inputs.");
}

void loop() {
  // 1. Monitor Keypad Input
  char key = keypad.getKey();
  if (key) {
    if (key == '#') {
      if (keypadInput == SECRET_PIN) {
        unlockDoor("KEYPAD");
      } else {
        Serial.println("Keypad PIN incorrect!");
        lcd.setCursor(0, 0);
        lcd.print("INVALID PIN!    ");
        playErrorChime();
        delay(1000);
      }
      keypadInput = ""; // Clear input buffer
    } 
    else if (key == '*') {
      keypadInput = ""; // Reset buffer
    } 
    else {
      if (keypadInput.length() < 4) {
        keypadInput += key;
      }
    }
  }
  
  // 2. Monitor Bluetooth Input (UART2)
  while (Serial2.available() > 0) {
    char c = Serial2.read();
    if (c == '\n' || c == '\r') {
      bluetoothInput.trim();
      if (bluetoothInput.length() > 0) {
        if (bluetoothInput == BT_UNLOCK_CMD) {
          unlockDoor("BLUETOOTH");
        } else {
          Serial.print("Unknown BT Command: "); Serial.println(bluetoothInput);
          playErrorChime();
        }
        bluetoothInput = ""; // Clear buffer
      }
    } else {
      bluetoothInput += c;
    }
  }
  
  // 3. Update LCD Idle screen
  lcd.setCursor(0, 0);
  lcd.print("Door: SECURE    ");
  lcd.setCursor(0, 1);
  lcd.print("PIN: ");
  for (int i = 0; i < keypadInput.length(); i++) lcd.print("*");
  for (int i = keypadInput.length(); i < 4; i++) lcd.print(" ");
  
  delay(20);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Keypad**, **HC-05**, **Servo**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire Keypad rows/cols, HC-05 RX/TX to **GPIO16/GPIO17**, Servo to **GPIO13**, Buzzer to **GPIO4**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Type `4321#` on the keypad widget. Watch the servo deadbolt open for 4 seconds.
5. Send the text `UNLOCK\n` via the Bluetooth terminal. Watch the servo open again.

## Expected Output
Serial Monitor:
```
Hybrid Smart Lock online. Ready for inputs.
Door Unlocked by: KEYPAD
Door Auto-Relocked.
Door Unlocked by: BLUETOOTH
Door Auto-Relocked.
```

LCD Display (idle):
```
Door: SECURE
PIN: ****
```

## Expected Canvas Behavior
* At boot, LCD shows "Door: SECURE". Servo is at 0°.
* Typing `4321#` on the keypad widget sweeps the servo to 90° and updates the LCD to "ACCESS GRANTED / By: KEYPAD".
* Sending the text `UNLOCK` over the Bluetooth Serial terminal also sweeps the servo and shows "ACCESS GRANTED / By: BLUETOOTH".
* In either case, the servo automatically returns to 0° after 4 seconds.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `keypadInput == SECRET_PIN` | Verifies the physical PIN code typed on the local keypad. |
| `Serial2.available() > 0` | Checks for incoming data packets from the Bluetooth module. |
| `bluetoothInput == BT_UNLOCK_CMD` | Verifies the remote Bluetooth serial command packet. |

## Hardware & Safety Concept: Hybrid Access Systems
Smart locks often combine local entry (keypad) with remote entry (Bluetooth or WiFi). In a hybrid system:
1. **Local keypad**: Provides entry if a phone battery is dead or Bluetooth connection fails.
2. **Bluetooth link**: Allows hands-free entry as the user approaches the door.
Using independent inputs on the same control loop ensures reliability.

## Try This! (Challenges)
1. **Master PIN changer**: Implement a keypad sequence (e.g. `*9999#`) that allows changing the entry PIN code.
2. **Bluetooth telemetry feed**: Broadcast lock status messages ("LOCKED" / "UNLOCKED") over Bluetooth.
3. **Lockout Protection Alarm**: Log incorrect keypad attempts, locking out the system and sounding a siren (GPIO 4) if three consecutive attempts fail.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Keypad buttons don't register | Keypad rows/columns miswired | Check keypad pinout connections and verify GPIO assignments 12–15 and 16, 17, 25, 26 |
| Bluetooth commands ignored | Missing line ending characters | Make sure the Bluetooth terminal is sending `\n` or `\r` line endings with commands |
| Servo does not move | Current draw overload | Power the servo from the 5V Vin rail, not the 3.3V rail |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [158 - ESP32 Keypad RFID Multi-factor Lock](158-esp32-keypad-rfid-multi-factor-lock.md)
- [120 - ESP32 Bluetooth DHT22 Telemetry Broadcast](../intermediate/120-esp32-bluetooth-dht22-telemetry-broadcast.md)
- [127 - ESP32 Bluetooth Remote Control Robot](../intermediate/127-esp32-bluetooth-remote-control-robot.md)
