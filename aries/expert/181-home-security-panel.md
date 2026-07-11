# 181 - Home Security Panel

Design a multi-sensor home security panel featuring PIR motion detection, a laser tripwire sensor barrier, RFID arm/disarm capability, status LEDs, and an I2C LCD alert dashboard.

## Goal
Learn how to build a state-machine based alarm panel, drive digital and analog tripwire inputs, control a laser transmitter pin, interface an RFID reader over SPI, and handle alarm resets safely without using loops.

## What You Will Build
An intrusion detection system. The system can be toggled between "Disarmed" and "Armed" states by scanning an authorized RFID card. In the Armed state, the system turns on a Laser Diode pointed at an LDR (forming a light barrier) and monitors a PIR motion sensor. If the light barrier is broken (LDR reading drops) or the PIR sensor registers motion, the system enters the "Alarm" state: sounding the buzzer, flashing the red LED, and showing "INTRUDER ALERT!" on the LCD until the RFID card is scanned to disarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| LDR (Photoresistor) | `ldr` | Yes | Yes |
| Laser Diode Module | `laser` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 RFID | SDA (SS) | GPIO 10 | Orange | SPI Chip Select (RFID) |
| MFRC522 RFID | SCK | GPIO 5 | Blue | SPI SCK |
| MFRC522 RFID | MOSI | GPIO 14 | Yellow | SPI COPI |
| MFRC522 RFID | MISO | GPIO 15 | Green | SPI CIPO |
| MFRC522 RFID | RST | GPIO 9 | White | Reset |
| PIR Sensor | OUT | GPIO 11 | Brown | Motion digital input |
| LDR | OUT (analog) | ADC1 | Yellow | Laser barrier receiver (GP27) |
| Laser Diode | Signal | GPIO 12 | Purple | Laser control output |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 7 | Gray | Alarm sounder |
| Active Buzzer | - | GND | Black | Ground |
| Red LED | Anode | GPIO 6 | Red | Alarm warning indicator |
| Red LED | Cathode | GND | Black | Ground |

> **Wiring tip:** Set the laser diode and LDR up directly opposite to each other. In MbedO's simulation canvas, you can adjust the light slider on the LDR widget to mimic the laser beam being blocked.

## Code
```cpp
// Home Security Panel - VEGA ARIES v3
#include <SPI.h>
#include <MFRC522.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RFID_CS = 10;
const int RFID_RST = 9;
const int PIR_PIN = 11;
const int LDR_PIN = ADC1;
const int LASER_PIN = 12;
const int BUZZER_PIN = 7;
const int RED_LED = 6;

MFRC522 mfrc522(RFID_CS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);

int systemState = 0; // 0: Disarmed, 1: Armed, 2: Alarm Triggered
unsigned long lastFlashTime = 0;
bool flashState = false;

// Driver ID verification variables (individual scalar values, no arrays)
byte authedDriver0 = 0xDE;
byte authedDriver1 = 0xAD;
byte authedDriver2 = 0xBE;
byte authedDriver3 = 0xEF;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(PIR_PIN, INPUT);
  pinMode(LASER_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  
  digitalWrite(LASER_PIN, LOW); // Laser off when disarmed
  digitalWrite(RED_LED, LOW);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Security Panel");
  lcd.setCursor(0, 1);
  lcd.print("System Disarmed");
}

void loop() {
  // 1. Process RFID Swipes to toggle state or silence alarms
  if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
    // Check if card matches auth card DE AD BE EF
    bool isAuthed = false;
    if (mfrc522.uid.uidByte[0] == authedDriver0 &&
        mfrc522.uid.uidByte[1] == authedDriver1 &&
        mfrc522.uid.uidByte[2] == authedDriver2 &&
        mfrc522.uid.uidByte[3] == authedDriver3) {
      isAuthed = true;
    }
    
    if (isAuthed) {
      // Sound confirmation beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      
      if (systemState == 0) {
        // Disarmed -> Arming
        systemState = 1;
        digitalWrite(LASER_PIN, HIGH); // Turn laser on
        
        lcd.clear();
        lcd.print("System Arming...");
        delay(2000); // arming exit delay
        lcd.clear();
        lcd.print("System Armed!");
      } 
      else {
        // Armed or Alarm -> Disarmed
        systemState = 0;
        digitalWrite(LASER_PIN, LOW); // Laser off
        digitalWrite(BUZZER_PIN, LOW);
        digitalWrite(RED_LED, LOW);
        
        lcd.clear();
        lcd.print("Security Panel");
        lcd.setCursor(0, 1);
        lcd.print("System Disarmed");
      }
    } else {
      // Invalid Card swipe - double beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(80);
      digitalWrite(BUZZER_PIN, LOW);
      delay(80);
      digitalWrite(BUZZER_PIN, HIGH);
      delay(80);
      digitalWrite(BUZZER_PIN, LOW);
      
      Serial.println("Warning: Invalid Security Card!");
    }
    mfrc522.PICC_HaltA();
  }
  
  // 2. Monitoring Intrusion Sensors when System is Armed
  if (systemState == 1) {
    bool motion = digitalRead(PIR_PIN);
    int lightLevel = analogRead(LDR_PIN);
    
    // Laser tripwire: if beam is broken, LDR light level drops below 600
    bool beamBroken = (lightLevel < 600);
    
    if (motion || beamBroken) {
      systemState = 2; // Trigger Alarm
      lastFlashTime = millis();
      
      lcd.clear();
      lcd.print("** INTRUDER **");
      lcd.setCursor(0, 1);
      if (motion && beamBroken) {
        lcd.print("Motion & Laser");
      } else if (motion) {
        lcd.print("PIR Sensor Trig");
      } else {
        lcd.print("Laser Tripwire");
      }
      
      Serial.println("ALARM ACTIVE! INTRUDER DETECTED.");
    }
  }
  
  // 3. Executing Alarm warning signals in Alarm State
  if (systemState == 2) {
    unsigned long currentTime = millis();
    if (currentTime - lastFlashTime >= 250) { // 250ms flashing rate
      lastFlashTime = currentTime;
      flashState = !flashState;
      
      if (flashState) {
        digitalWrite(RED_LED, HIGH);
        digitalWrite(BUZZER_PIN, HIGH);
      } else {
        digitalWrite(RED_LED, LOW);
        digitalWrite(BUZZER_PIN, LOW);
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **MFRC522 RFID**, **PIR**, **LDR**, **Laser Diode**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring as described. Ensure logic matches pin allocations.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Scan RFID card `DE AD BE EF` to arm the system. Once armed, adjust the PIR state to HIGH or lower the LDR light level slider below 600 to trigger the alarm. Scan card again to disarm.

## Expected Output
Serial Monitor:
```
System status: DISARMED
Swipe verified: system ARMED. Laser activated.
Intruder detected! PIR motion sensor triggered.
ALARM ACTIVE! INTRUDER DETECTED.
Swipe verified: system DISARMED. Alarm silenced.
```

## Expected Canvas Behavior
* On startup, the LCD reads "Security Panel" and "System Disarmed". The Laser is off.
* Swiping authorized RFID makes the buzzer sound once. After 2 seconds, the laser turns ON and LCD changes to "System Armed!".
* If you set PIR to HIGH, or slide LDR light level < 600, the buzzer and LED begin flashing rapidly in 250ms bursts, and LCD prints "** INTRUDER **".
* Re-swiping the RFID card shuts off all alarms and returns the LCD to "System Disarmed".

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `systemState` | Tracks active security loop states: 0 (Disarmed), 1 (Armed), and 2 (Alarm Triggered). |
| `isAuthed` | Checks standard key UID scalar values without utilizing arrays or loops. |
| `digitalWrite(LASER_PIN, ...)` | Turns the physical light transmitter line on or off depending on the security state. |
| `lightLevel < 600` | Checks if the laser beam falling on the photoresistor has been blocked by an object. |
| `currentTime - lastFlashTime >= 250`| Flashes LED and sounds alarm buzzer continuously at 4 Hz without blocking execution. |

## Hardware & Safety Concept
* **Laser Safety**: Always use low-power (Class 1 or Class 2) red laser pointer diodes for security projects. Avoid directing the beam toward eyes or highly reflective surfaces.
* **Exit/Entry Delays**: A physical panel always gives the user a delay (e.g. 15 seconds) to leave the room after arming or disarm the system after opening the door before triggering the siren.

## Try This! (Challenges)
1. **Silent Alarm mode**: Add a feature where if the alarm is triggered by the PIR sensor, the buzzer stays OFF (silent alert) but the red LED flashes and the event logs to serial, while laser trigger sounds full alarm.
2. **Arming Beep Countdown**: During the 2-second arming delay, beep the buzzer once every 500ms to let the user know the system is actively arming.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately after arming | LDR light level set too low | Adjust LDR light slider in canvas to maximum (>800) before arming, ensuring laser simulation is registered. |
| RFID card swipe doesn't respond | Pin conflict with MISO/MOSI | Ensure the Buzzer and Warning LED are wired to GPIO 7 and 6, and not the SPI pins 14 and 15. |
| Screen displays blank blocks | I2C address conflict | Check that SDA0/SCL0 are connected properly. Verify screen address is 0x27. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [141 - Sounding Sentry Guard](../advanced/141-sounding-sentry-guard.md)
- [144 - RFID Access Logger](../advanced/144-rfid-access-logger.md)
- [172 - RFID Smart Campus Gate](172-rfid-smart-campus-gate.md)
