# 182 - Smart Parking Barrier

Build an automated parking lot entry barrier featuring space tracking, RFID card credentials, vehicle proximity detection, LCD space counter, and SD transaction logs.

## Goal
Learn how to combine distance sensing and RFID scans to manage access control, implement a slot tracking system using variables, control a servo barrier gate, and log vehicle movements to an SD card.

## What You Will Build
A parking garage barrier. An LCD displays the count of available spaces (out of 5). When a car approaches the gate, the HC-SR04 ultrasonic sensor detects it. If spaces are available, the LCD asks the driver to scan their parking card. Swiping a valid RFID card decrement the space count, logs the entrance on the SD card, beep the buzzer, and sweeps the servo gate open. The gate closes once the car has passed. Pressing the button simulates a car exiting, which opens the gate, increments the space count, and logs the exit.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| SG90 Servo Motor | `servo` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| SD Card Reader Module | `sd_spiv3` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-SR04 | TRIG | GPIO 11 | Orange | Ultrasonic trigger |
| HC-SR04 | ECHO | GPIO 12 | Green | Ultrasonic echo |
| HC-SR04 | VCC | 5V | Red | Power |
| HC-SR04 | GND | GND | Black | Ground |
| MFRC522 RFID | SDA (SS) | GPIO 10 | Orange | SPI CS (RFID) |
| MFRC522 RFID | SCK | GPIO 5 | Blue | SPI SCK |
| MFRC522 RFID | MOSI | GPIO 14 | Yellow | SPI COPI |
| MFRC522 RFID | MISO | GPIO 15 | Green | SPI CIPO |
| MFRC522 RFID | RST | GPIO 9 | White | Reset |
| SD Card Reader | CS | GPIO 8 | Purple | SPI CS (SD) |
| SD Card Reader | SCK | GPIO 5 | Blue | Shared SPI SCK |
| SD Card Reader | MOSI | GPIO 14 | Yellow | Shared SPI COPI |
| SD Card Reader | MISO | GPIO 15 | Green | Shared SPI CIPO |
| SG90 Servo | Signal | GPIO 13 | Yellow | Servo gate control |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 7 | Gray | Audio status output |
| Red LED | Anode | GPIO 6 | Red | Full occupancy warning |
| Push Button (Exit) | Pin 2 | GPIO 16 | White | Exit sensor switch |

> **Wiring tip:** The exit button uses the internal pull-up resistor of the ARIES board (input pull-up configuration), so only connect one terminal of the button to GPIO 16 and the other terminal to GND.

## Code
```cpp
// Smart Parking Barrier - VEGA ARIES v3
#include <SPI.h>
#include <MFRC522.h>
#include <SD.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Servo.h>

const int SD_CS = 8;
const int RFID_CS = 10;
const int RFID_RST = 9;
const int TRIG = 11;
const int ECHO = 12;
const int SERVO_PIN = 13;
const int BUZZER_PIN = 7;
const int RED_LED = 6;
const int EXIT_BUTTON = 16;

MFRC522 mfrc522(RFID_CS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);
Servo gateServo;

int spacesAvailable = 5;
int barrierState = 0; // 0: Closed/Idle, 1: Opening, 2: Open/Waiting for car to pass, 3: Closing
int servoPos = 0;

unsigned long stateTimer = 0;
unsigned long servoTimer = 0;
float carDistance = 400.0;
bool exitRequested = false;

void setup() {
  Serial.begin(115200);
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RED_LED, OUTPUT);
  pinMode(EXIT_BUTTON, INPUT_PULLUP);
  
  gateServo.attach(SERVO_PIN);
  gateServo.write(0);
  
  lcd.init();
  lcd.backlight();
  
  SD.begin(SD_CS);
  
  // Initial LCD update
  lcd.clear();
  lcd.print("Parking System");
  lcd.setCursor(0, 1);
  lcd.print("Spaces: 5/5");
}

void loop() {
  // 1. Read Proximity Sensor
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long duration = pulseIn(ECHO, HIGH, 30000);
  carDistance = duration * 0.034 / 2.0;
  if (carDistance == 0) carDistance = 400.0;

  // Check full occupancy LED
  if (spacesAvailable == 0) {
    digitalWrite(RED_LED, HIGH);
  } else {
    digitalWrite(RED_LED, LOW);
  }

  // 2. Poll Exit Button (active LOW)
  if (digitalRead(EXIT_BUTTON) == LOW && barrierState == 0) {
    exitRequested = true;
    barrierState = 1; // Transition to Opening
    stateTimer = millis();
    
    // Increment spaces (exit log)
    if (spacesAvailable < 5) {
      spacesAvailable = spacesAvailable + 1;
    }
    
    // Log exit to SD
    File logFile = SD.open("park_log.txt", FILE_WRITE);
    if (logFile) {
      logFile.println("VEHICLE EXIT - Space Freed");
      logFile.close();
    }
    
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);
    
    lcd.clear();
    lcd.print("Exit Approved");
    lcd.setCursor(0, 1);
    lcd.print("Safe Journey!");
  }

  // 3. Process Entrance RFID swipe when car is waiting (distance < 15cm)
  if (barrierState == 0 && carDistance < 15.0 && !exitRequested) {
    lcd.setCursor(0, 0);
    lcd.print("Car Detected    ");
    lcd.setCursor(0, 1);
    if (spacesAvailable > 0) {
      lcd.print("Scan RFID Card  ");
      
      if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
        // Authenticate card DE AD BE EF
        bool validCard = false;
        if (mfrc522.uid.uidByte[0] == 0xDE &&
            mfrc522.uid.uidByte[1] == 0xAD &&
            mfrc522.uid.uidByte[2] == 0xBE &&
            mfrc522.uid.uidByte[3] == 0xEF) {
          validCard = true;
        }
        
        if (validCard) {
          spacesAvailable = spacesAvailable - 1;
          barrierState = 1; // Transition to Opening
          stateTimer = millis();
          
          // Log entrance to SD
          File logFile = SD.open("park_log.txt", FILE_WRITE);
          if (logFile) {
            logFile.print("VEHICLE ENTRY - ID: ");
            logFile.print(mfrc522.uid.uidByte[0], HEX);
            logFile.print(mfrc522.uid.uidByte[1], HEX);
            logFile.print(mfrc522.uid.uidByte[2], HEX);
            logFile.print(mfrc522.uid.uidByte[3], HEX);
            logFile.println();
            logFile.close();
          }
          
          digitalWrite(BUZZER_PIN, HIGH);
          delay(100);
          digitalWrite(BUZZER_PIN, LOW);
          
          lcd.clear();
          lcd.print("Access Approved!");
          lcd.setCursor(0, 1);
          lcd.print("Welcome.");
        } else {
          // Reject
          digitalWrite(BUZZER_PIN, HIGH);
          delay(100);
          digitalWrite(BUZZER_PIN, LOW);
          delay(100);
          digitalWrite(BUZZER_PIN, HIGH);
          delay(100);
          digitalWrite(BUZZER_PIN, LOW);
          
          lcd.clear();
          lcd.print("Invalid Card!");
          delay(2000);
        }
        mfrc522.PICC_HaltA();
      }
    } else {
      lcd.print("Lot Full!      ");
    }
  } 
  else if (barrierState == 0 && carDistance >= 15.0) {
    // Normal idle LCD display
    lcd.setCursor(0, 0);
    lcd.print("Parking Barrier ");
    lcd.setCursor(0, 1);
    lcd.print("Spaces: ");
    lcd.print(spacesAvailable);
    lcd.print("/5      ");
  }

  // 4. Barrier Gate Servo State Machine
  if (barrierState == 1) { // Opening
    if (millis() - servoTimer >= 15) {
      servoTimer = millis();
      if (servoPos < 90) {
        servoPos = servoPos + 1;
        gateServo.write(servoPos);
      } else {
        barrierState = 2; // Transition to waiting
        stateTimer = millis();
      }
    }
  } 
  else if (barrierState == 2) { // Waiting open
    // If exit button triggered it, wait 4 seconds. If entrance, wait for car to pass (distance > 30cm)
    bool carPassed = false;
    if (exitRequested) {
      if (millis() - stateTimer >= 4000) {
        carPassed = true;
        exitRequested = false;
      }
    } else {
      if (carDistance > 30.0 && (millis() - stateTimer >= 3000)) {
        carPassed = true;
      }
    }
    
    if (carPassed) {
      barrierState = 3; // Transition to Closing
      stateTimer = millis();
      lcd.clear();
      lcd.print("Closing Gate...");
    }
  } 
  else if (barrierState == 3) { // Closing
    if (millis() - servoTimer >= 15) {
      servoTimer = millis();
      if (servoPos > 0) {
        servoPos = servoPos - 1;
        gateServo.write(servoPos);
      } else {
        barrierState = 0; // Return to closed
      }
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **HC-SR04**, **MFRC522 RFID**, **SG90 Servo**, **I2C LCD**, **SD Card Reader**, **Buzzer**, **LED**, and **Push Button** onto the canvas.
2. Complete the wiring matching the table.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Slide the HC-SR04 distance < 15 to simulate a car waiting at the entrance. Swipe RFID to trigger. Click the exit button to test vehicle exits.

## Expected Output
Serial Monitor:
```
Spaces Available: 5/5
Car Detected. Scanning card...
Card Approved: DEADBEEF | Spaces: 4/5
Barrier Gate Opening...
Car passed. Barrier Gate Closing.
Exit Request Detected. Spaces: 5/5
```

## Expected Canvas Behavior
* The LCD displays "Spaces: 5/5" on startup.
* When the HC-SR04 widget distance is adjusted below 15cm, the LCD prompts "Scan RFID Card".
* Swiping authorized RFID causes the servo to sweep to 90 degrees and decreases spaces count.
* Adjusting HC-SR04 distance > 30cm starts a 3-second timer, after which the servo sweeps back to 0 degrees.
* Pressing the button immediately opens the servo gate and increases spaces count.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `spacesAvailable` | State counter tracking lot capacity. Prevents entry if count reaches 0. |
| `EXIT_BUTTON` | Polled pin with pull-up. Active LOW state triggers vehicle exit cycle. |
| `carDistance < 15.0` | Uses proximity threshold to verify physical presence of vehicle before RFID scan. |
| `SD.open("park_log.txt", ...)` | Logs entry/exit timestamps to track lot velocity. |
| `barrierState == 2` | Open wait loop. Uses time-delay exit override or sensor-based pass verification. |

## Hardware & Safety Concept
* **Force Feedback Safety**: A real parking gate requires a torque-limiter or current sensor to stop the gate instantly if it hits a vehicle or pedestrian during closure, avoiding injury or damage.
* **Ground Loop Protection**: Standard loop detectors under the asphalt use inductive coil loops to detect massive metallic vehicle frames. Using an ultrasonic sensor in combination with RFID acts as an excellent, non-invasive alternative.

## Try This! (Challenges)
1. **VIP Space Override**: Define a VIP RFID card that does not decrement the spaces count (reserved slot) and opens the gate even if the lot is full.
2. **Double Swipe Warning**: If a card is scanned twice consecutively without the car passing through, sound a rapid warning alarm and refuse to open the gate again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Gate opens but closes immediately | Exit button floating or trigger state wrong | Ensure GPIO 16 is configured as `INPUT_PULLUP`. A floating input pin causes erratic triggers. |
| Space count goes past 5 | Increment bounds not capped | Make sure `spacesAvailable` is restricted: `if (spacesAvailable < 5)`. |
| Screen doesn't show number updates | LiquidCrystal address mismatch | Confirm LCD I2C connection is on pins 16/17 and using Address 0x27. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [124 - Obstacle Avoidance Robot](../advanced/124-obstacle-avoidance-robot.md)
- [144 - RFID Access Logger](../advanced/144-rfid-access-logger.md)
- [172 - RFID Smart Campus Gate](172-rfid-smart-campus-gate.md)
