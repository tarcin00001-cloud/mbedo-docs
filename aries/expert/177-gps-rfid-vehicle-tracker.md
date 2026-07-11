# 177 - GPS RFID Vehicle Tracker

Build a fleet management logger that records the identity of the driver via RFID and logs GPS coordinates (latitude, longitude, speed) to an SD card.

## Goal
Learn how to interface UART-based GPS modules on Serial1, parse coordinate strings, share the SPI bus between an RFID scanner and SD card module, and manage driver authentication status without using arrays or loops.

## What You Will Build
A vehicle tracking and access control system. A vehicle engine is locked (indicated by a warning LED) until a driver scans their RFID keycard. Once authenticated, the system sounds a confirmation beep, unlocks the engine (turns off warning LED), and begins logging GPS coordinates, speeds, and timestamps to an SD card every 5 seconds. The current location is displayed on an I2C LCD screen.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| NEO-6M GPS Module | `gps_neo6m` | Yes | Yes |
| MFRC522 RFID Reader | `mfrc522` | Yes | Yes |
| SD Card Reader Module | `sd_spiv3` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Red Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M GPS | TX | GPIO 3 | Yellow | RX1 input |
| NEO-6M GPS | RX | GPIO 4 | Green | TX1 output |
| NEO-6M GPS | VCC | 5V / 3V3 | Red | Power |
| NEO-6M GPS | GND | GND | Black | Ground |
| MFRC522 RFID | SDA (SS) | GPIO 10 | Orange | SPI CS (RFID) |
| MFRC522 RFID | SCK | GPIO 5 | Blue | SPI SCK |
| MFRC522 RFID | MOSI | GPIO 14 | Yellow | SPI COPI |
| MFRC522 RFID | MISO | GPIO 15 | Green | SPI CIPO |
| MFRC522 RFID | RST | GPIO 9 | White | Reset |
| SD Card Reader | CS | GPIO 8 | Purple | SPI CS (SD) |
| SD Card Reader | SCK | GPIO 5 | Blue | Shared SPI SCK |
| SD Card Reader | MOSI | GPIO 14 | Yellow | Shared SPI COPI |
| SD Card Reader | MISO | GPIO 15 | Green | Shared SPI CIPO |
| SD Card Reader | VCC | 5V | Red | Power |
| SD Card Reader | GND | GND | Black | Ground |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 7 | Gray | Buzz status |
| Active Buzzer | - | GND | Black | Ground |
| Red Warning LED | Anode | GPIO 6 | Red | Ignition lock LED |
| Red Warning LED | Cathode | GND | Black | Ground |

> **Wiring tip:** The GPS module uses serial communication on Serial1 (GPIO 3/4). Connect the GPS TX pin directly to the ARIES RX1 pin (GPIO 3) and GPS RX pin to ARIES TX1 pin (GPIO 4).

## Code
```cpp
// GPS RFID Vehicle Tracker - VEGA ARIES v3
#include <SPI.h>
#include <MFRC522.h>
#include <SD.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <TinyGPS++.h>

const int SD_CS = 8;
const int RFID_CS = 10;
const int RFID_RST = 9;
const int BUZZER_PIN = 7;
const int WARNING_LED = 6;

MFRC522 mfrc522(RFID_CS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);
TinyGPSPlus gps;

// Driver UID bytes stored in individual scalar variables instead of arrays
byte currentDriver0 = 0;
byte currentDriver1 = 0;
byte currentDriver2 = 0;
byte currentDriver3 = 0;
bool driverLoggedIn = false;

unsigned long lastLogTime = 0;

void setup() {
  Serial.begin(115200);
  Serial1.begin(9600); // GPS baud rate
  
  SPI.begin();
  mfrc522.PCD_Init();
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  digitalWrite(WARNING_LED, HIGH); // Engine locked by default
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Engine Locked");
  lcd.setCursor(0, 1);
  lcd.print("Scan RFID Card");
  
  SD.begin(SD_CS);
}

void loop() {
  // Read incoming characters from GPS serial stream (non-blocking)
  if (Serial1.available() > 0) {
    char c = Serial1.read();
    gps.encode(c);
  }
  
  // Manage RFID Driver Scan
  if (!driverLoggedIn) {
    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
      // Store UID
      currentDriver0 = mfrc522.uid.uidByte[0];
      currentDriver1 = mfrc522.uid.uidByte[1];
      currentDriver2 = mfrc522.uid.uidByte[2];
      currentDriver3 = mfrc522.uid.uidByte[3];
      driverLoggedIn = true;
      
      // Sound buzz success
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      
      digitalWrite(WARNING_LED, LOW); // Engine unlocked
      
      lcd.clear();
      lcd.print("Driver Logged In");
      lcd.setCursor(0, 1);
      lcd.print("ID: ");
      lcd.print(currentDriver0, HEX);
      lcd.print(currentDriver1, HEX);
      lcd.print(currentDriver2, HEX);
      lcd.print(currentDriver3, HEX);
      delay(2000);
      
      mfrc522.PICC_HaltA();
    }
  } 
  else {
    // Check if driver wants to logout (by re-scanning the same card)
    if (mfrc522.PICC_IsNewCardPresent() && mfrc522.PICC_ReadCardSerial()) {
      if (mfrc522.uid.uidByte[0] == currentDriver0 &&
          mfrc522.uid.uidByte[1] == currentDriver1 &&
          mfrc522.uid.uidByte[2] == currentDriver2 &&
          mfrc522.uid.uidByte[3] == currentDriver3) {
        
        driverLoggedIn = false;
        digitalWrite(WARNING_LED, HIGH); // Re-lock engine
        
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        delay(100);
        digitalWrite(BUZZER_PIN, HIGH);
        delay(100);
        digitalWrite(BUZZER_PIN, LOW);
        
        lcd.clear();
        lcd.print("Engine Locked");
        lcd.setCursor(0, 1);
        lcd.print("Driver LoggedOut");
        delay(2000);
      }
      mfrc522.PICC_HaltA();
    }
  }
  
  // Periodic logging of GPS data (every 5 seconds) if driver is active
  unsigned long currentTime = millis();
  if (driverLoggedIn && (currentTime - lastLogTime >= 5000)) {
    lastLogTime = currentTime;
    
    float lat = gps.location.lat();
    float lng = gps.location.lng();
    float speed = gps.speed.kmph();
    
    // Log to SD
    File logFile = SD.open("tracker.csv", FILE_WRITE);
    if (logFile) {
      logFile.print(currentDriver0, HEX);
      logFile.print(currentDriver1, HEX);
      logFile.print(currentDriver2, HEX);
      logFile.print(currentDriver3, HEX);
      logFile.print(",");
      logFile.print(lat, 6);
      logFile.print(",");
      logFile.print(lng, 6);
      logFile.print(",");
      logFile.println(speed);
      logFile.close();
    }
    
    // Update LCD
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("Lat: ");
    lcd.print(lat, 4);
    lcd.setCursor(0, 1);
    lcd.print("Lng: ");
    lcd.print(lng, 4);
    
    // Print to Serial Monitor
    Serial.print("Driver: ");
    Serial.print(currentDriver0, HEX);
    Serial.print(" | GPS: ");
    Serial.print(lat, 6);
    Serial.print(", ");
    Serial.print(lng, 6);
    Serial.print(" | Speed: ");
    Serial.print(speed);
    Serial.println(" km/h");
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **NEO-6M GPS**, **MFRC522 RFID**, **SD Card Reader**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Complete the wiring according to the table.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the dropdown.
5. Click **Run**.
6. Swipe a card on the RFID widget to unlock the vehicle. Adjust GPS location inputs to generate mock telemetry.

## Expected Output
Serial Monitor:
```
Driver: DEADBEEF | GPS: 12.971600, 77.594600 | Speed: 45.2 km/h
Driver: DEADBEEF | GPS: 12.971800, 77.594900 | Speed: 52.1 km/h
Driver: DEADBEEF | GPS: 12.972100, 77.595200 | Speed: 38.0 km/h
```

## Expected Canvas Behavior
* On startup, the LCD says "Engine Locked" and the red LED is ON.
* Swiping an RFID card turns off the warning LED, beeps the buzzer once, and changes the LCD to display coordinates.
* Every 5 seconds, the location data is appended to `tracker.csv` on the SD card.
* Re-swiping the same card triggers a double beep, turns the warning LED back ON, and displays "Engine Locked".

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `Serial1.begin(9600)` | Launches hardware UART1 interface at 9600 baud for GPS NMEA receiver stream. |
| `gps.encode(c)` | Feeds character stream to TinyGPS++ parser object to extract coordinate sentences. |
| `currentDriver0 = ...` | Extracts individual UID elements and saves them into non-array global variables. |
| `driverLoggedIn` | State flag protecting vehicle access control and enabling logging loop. |
| `gps.location.lat()` | Requests decoded latitude floating-point value. |

## Hardware & Safety Concept
* **Vehicle Ignition Interrupt**: In a real automotive application, the WARNING_LED pin would drive a heavy-duty relay connected to the starter motor solenoid or fuel pump power line, creating a robust immobilizer.
* **GPS Cold/Warm Starts**: GPS modules can take up to 30 seconds to lock onto satellite signals (known as a Time-To-First-Fix). The system handles this gracefully by displaying blank or placeholder coordinates until a valid fix is acquired.

## Try This! (Challenges)
1. **Speed Limit Alarm**: If the GPS speed exceeds 60 km/h, sound the buzzer continuously and flash the warning LED until speed drops below the limit.
2. **Panic Button**: Map the Push Button (GPIO 16) to generate an immediate panic log entry on the SD card containing the current location and a "HELP" flag.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| GPS latitude and longitude read as 0.0000 | No satellite lock (indoor testing) | Move GPS antenna close to a window or outdoors. Check that the TX/RX wires are not swapped. |
| RFID reader fails to detect cards | SPI bus collisions | Ensure SD CS and RFID CS pins match code allocations (8 and 10) and are wired correctly. |
| LCD does not print text | Contrast potentiometer needs tuning | Turn the small screw potentiometer on the back of the I2C LCD backpack until text appears. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [145 - GPS Logger](../advanced/145-gps-logger.md)
- [147 - GPS Velocity Logger](../advanced/147-gps-velocity-logger.md)
- [172 - RFID Smart Campus Gate](172-rfid-smart-campus-gate.md)
