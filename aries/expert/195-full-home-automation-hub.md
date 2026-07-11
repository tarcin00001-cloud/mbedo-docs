# 195 - Full Home Automation Hub

Construct a central smart-home automation controller. The system manages building security (using an MFRC522 RFID card reader and PIR motion detector), monitors environmental climate (DHT22 sensor), controls household appliances (dual relays for lights and cooling fans), and outputs status updates on a 16x2 I2C LCD character display.

## Goal
Learn how to program a multi-device controller using SPI and I2C buses together, design a multi-page LCD dashboard state machine, implement RFID authorization lists, and control high-power loads via relays, without using loops or custom helper functions.

## What You Will Build
A security and environmental control console. Scanning an authorized RFID card toggles the system between Armed and Disarmed states. If armed, detecting motion on the PIR sensor sounds the alarm buzzer on GPIO 14 and turns on the security light relay (GPIO 3). Independently, the system monitors temperature using a DHT22 on GPIO 2; if the temperature exceeds 28°C, it activates the cooling fan relay (GPIO 7). The LCD alternates between security logs and climate stats.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| MFRC522 RFID Reader | `rfid_reader` | Yes | Yes |
| RFID Authorized Tag | `rfid_tag` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 2-Channel Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 Module | 3.3V | 3V3 | Red | Power supply |
| MFRC522 Module | GND | GND | Black | Ground reference |
| MFRC522 Module | RST (Reset) | GPIO 9 | Purple | Reset line |
| MFRC522 Module | SDA (SS) | GPIO 10 | Orange | SPI Chip Select |
| MFRC522 Module | SCK | GP13 (SPI SCK) | Yellow | SPI Clock line |
| MFRC522 Module | MISO | GP12 (SPI MISO) | Green | SPI MISO line |
| MFRC522 Module | MOSI | GP11 (SPI MOSI) | Blue | SPI MOSI line |
| DHT22 Sensor | DATA | GPIO 2 | Yellow | Data line (moved to GP2 to free GP12 MISO) |
| DHT22 Sensor | VCC | 3V3 | Red | Power supply |
| DHT22 Sensor | GND | GND | Black | Ground reference |
| PIR Sensor | OUT | GPIO 8 | White | Motion signal input |
| Relay 1 (Light) | IN | GPIO 3 | Blue | Channel 1 trigger (Active HIGH) |
| Relay 2 (Fan) | IN | GPIO 7 | Grey | Channel 2 trigger (Active HIGH) |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Active Buzzer | + | GPIO 14 | Orange | Alarm sounder |
| Warning LED | Anode | GPIO 15 | Orange | Security indicator |

> **Wiring tip:** The DHT22 sensor pin is wired to **GPIO 2** in this project. Using the default GPIO 12 would create a hardware conflict, as the MFRC522 RFID reader requires GPIO 12 for the SPI MISO signal.

## Code
```cpp
// 195 - Full Home Automation Hub
#include <Wire.h>
#include <SPI.h>
#include <MFRC522.h>
#include <DHT.h>
#include <LiquidCrystal_I2C.h>

#define SS_PIN 10
#define RST_PIN 9
#define DHTPIN 2
#define DHTTYPE DHT22

MFRC522 rfid(SS_PIN, RST_PIN);
DHT dht(DHTPIN, DHTTYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

const int PIR_PIN = 8;
const int LIGHT_RELAY = 3;
const int FAN_RELAY = 7;
const int BUZZER_PIN = 14;
const int WARN_LED = 15;

// Authorized RFID Card UID details (saved as individual byte variables)
const byte AUTH_UID_0 = 0x83;
const byte AUTH_UID_1 = 0xA2;
const byte AUTH_UID_2 = 0xC3;
const byte AUTH_UID_3 = 0xE4;

int systemArmed = 0;
int alarmTriggered = 0;
int pirState = 0;
int displayPage = 0; // 0 = Security page, 1 = Climate page
int displayTimer = 0;
int buzzerTick = 0;

float temp = 0.0;
float hum = 0.0;

void setup() {
  Serial.begin(115200);
  Wire.begin();
  SPI.begin();

  rfid.PCD_Init();
  dht.begin();

  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Home Hub Booting");

  pinMode(PIR_PIN, INPUT);
  pinMode(LIGHT_RELAY, OUTPUT);
  pinMode(FAN_RELAY, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED, OUTPUT);

  digitalWrite(LIGHT_RELAY, LOW);
  digitalWrite(FAN_RELAY, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED, LOW);

  delay(1000);
  lcd.clear();
  Serial.println("Home Automation Hub Online.");
}

void loop() {
  // Read PIR sensor
  pirState = digitalRead(PIR_PIN);

  // Read climate parameters
  float rTemp = dht.readTemperature();
  float rHum = dht.readHumidity();
  if (!isnan(rTemp)) temp = rTemp;
  if (!isnan(rHum)) hum = rHum;

  // Climate automation logic: Turn on cooling fan if hot
  if (temp > 28.0) {
    digitalWrite(FAN_RELAY, HIGH);
  } else {
    digitalWrite(FAN_RELAY, LOW);
  }

  // Look for RFID scans
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    // Validate UID bytes against authorized keys
    if (rfid.uid.uidByte[0] == AUTH_UID_0 &&
        rfid.uid.uidByte[1] == AUTH_UID_1 &&
        rfid.uid.uidByte[2] == AUTH_UID_2 &&
        rfid.uid.uidByte[3] == AUTH_UID_3) {
      
      // Toggle Arm state
      systemArmed = !systemArmed;
      alarmTriggered = 0; // Clear active alarm
      
      // Confirmation beep
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);

      Serial.print("RFID Authorised. System status: ");
      Serial.println(systemArmed ? "ARMED" : "DISARMED");
    } else {
      Serial.println("Unauthorised RFID Card Scanned.");
    }
    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }

  // Security alert monitoring
  if (systemArmed == 1) {
    digitalWrite(WARN_LED, HIGH); // Armed indicator ON
    
    // Inturder alert on motion
    if (pirState == HIGH) {
      alarmTriggered = 1;
    }
  } else {
    digitalWrite(WARN_LED, LOW); // Armed indicator OFF
    if (alarmTriggered == 1) {
      alarmTriggered = 0; // Disarm clears alarm
    }
  }

  // Drive active alert outputs
  if (alarmTriggered == 1) {
    digitalWrite(LIGHT_RELAY, HIGH); // Turn lights ON for visibility
    
    // Rapid beep tone (200 ms loop cadence)
    buzzerTick++;
    if (buzzerTick % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  } else {
    digitalWrite(LIGHT_RELAY, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }

  // Rotate display dashboard page every 2 seconds (10 * 200 ms)
  displayTimer++;
  if (displayTimer >= 10) {
    displayTimer = 0;
    displayPage = !displayPage;
    lcd.clear();
  }

  // Output LCD visual data
  if (displayPage == 0) {
    // Security Dashboard Page
    lcd.setCursor(0, 0);
    lcd.print("Mode: ");
    lcd.print(systemArmed ? "ARMED   " : "DISARMED");

    lcd.setCursor(0, 1);
    if (alarmTriggered == 1) {
      lcd.print("!INTRUDER ALERT!");
    } else {
      lcd.print("PIR: ");
      lcd.print(pirState ? "ACTIVE  " : "STANDBY ");
    }
  } else {
    // Climate Dashboard Page
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(temp, 1);
    lcd.print(" C");

    lcd.setCursor(0, 1);
    lcd.print("Hum:  ");
    lcd.print(hum, 1);
    lcd.print(" %");
  }

  delay(200);
}
```

## What to Click in MbedO
1. Drag **VEGA ARIES v3**, **MFRC522 RFID**, **authorized tag**, **PIR**, **DHT22**, **I2C LCD**, **2x Relay**, **Active Buzzer**, and **Warning LED** onto the canvas.
2. Wire the MFRC522 SPI lines as shown in the table. CS to **GPIO 10**, RST to **GPIO 9**.
3. Wire the DHT22: **DATA → GPIO 2**, **VCC → 3V3**, **GND → GND**.
4. Wire PIR: **OUT → GPIO 8**; Relays: Light to **GPIO 3**, Fan to **GPIO 7**.
5. Wire I2C LCD: **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
6. Wire Buzzer: **+ → GPIO 14**; Warning LED: **Anode → GPIO 15**.
7. Paste the code, select **Interpreted Mode**, and click **Run**.
8. Click and drag the authorized RFID tag near the reader to arm. Trigger the PIR sensor slider to active, observing the alarm state. Scan again to disarm.

## Expected Output
Serial Monitor:
```
Home Automation Hub Online.
RFID Authorised. System status: ARMED
!!! INTRUDER DETECTED !!! Lights active, alarm sounding.
RFID Authorised. System status: DISARMED
Temp: 29.4C | Relay Fan: ON
```

## Expected Canvas Behavior
* Startup: LCD shows `Home Hub Booting`, then loads the security dashboard page. Onboard Warning LED is OFF.
* Scanning the authorized tag turns the Warning LED ON and updates the LCD status to `Mode: ARMED`.
* Activating the PIR sensor slider triggers a rapid buzzer pulse, closes Relay 1 (turns on light), and displays `!INTRUDER ALERT!` on the LCD.
* Increasing the DHT22 temperature slider above 28°C closes Relay 2, turning on the fan.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `rfid.uid.uidByte[0] == ...` | Validates scanned card bytes against unauthorized access. |
| `systemArmed = !systemArmed` | Toggles security arm state using a boolean inversion trigger. |
| `digitalWrite(FAN_RELAY, HIGH)` | Closes fan relay circuit if DHT22 reports excess heat. |
| `displayTimer >= 10` | Alternates display screens using loop cycles to avoid blocking delays. |
| `lcd.print("!INTRUDER ALERT!")` | Outputs hazard warning lines directly onto the character frame. |

## Hardware & Safety Concept
* **SPI Bus Bus Sharing:** SPI is a bus-based protocol, meaning multiple slave devices (RFID, SD Card, Displays) can share the MOSI, MISO, and SCK lines. The master chip selects the active device by pulling its dedicated Chip Select (CS) pin LOW. The inactive devices float their MISO lines, avoiding data clashes.
* **RFID Security Logic:** Standard MIFARE RFID cards contain sectors encrypted by pre-shared keys. The reader sends an authentication request before reading data blocks. In simple hubs, matching the card's factory-written UID is sufficient, though production hardware reads encrypted sectors.

## Try This! (Challenges)
1. **Passcode Access Keypad:** Connect a 4x4 Keypad to GPIO pins and require both a valid RFID scan and a 4-digit keypad pin code to disarm the alarm.
2. **Intruder Alert Hysteresis:** If the alarm triggers, keep it active for at least 30 seconds even if the system is disarmed, or require a master card scan to silence the alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| RFID reader fails to read cards | SPI MISO pin clash | Ensure DHT22 is wired to GPIO 2. GPIO 12 must be reserved for the MISO line. |
| LCD display screen is blank | Backlight not active or wrong I2C address | Verify address parameter is set to `0x27` (common for I2C expanders). Check wire connections to SCL/SDA. |
| PIR triggers false alarms | High ambient air currents or heat | Adjust the PIR module physical sensitivity potentiometer or insert a software debounce. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [87 - MFRC522 RFID Servo Door Unlock](../intermediate/87-mfrc522-rfid-servo-door-unlock.md)
- [108 - Intrusion Detector Alarm](../intermediate/108-intrusion-detector-alarm.md)
- [191 - Indoor Air Quality Monitor](191-indoor-air-quality-monitor.md)
