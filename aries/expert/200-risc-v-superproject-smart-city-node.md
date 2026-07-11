# 200 - RISC-V Superproject: Smart City Node

The grand capstone of the VEGA ARIES v3 curriculum. Integrate a DHT22 climate sensor, MQ-2 air quality/smoke detector, HC-SR04 ultrasonic traffic/distance sensor, Neo-6M GPS tracking module, MFRC522 RFID reader, SPI SD card telemetry logger, and an I2C LCD dashboard to construct an autonomous RISC-V smart city infrastructure monitor.

## Goal
Master multi-bus hardware integration (SPI, I2C, UART, One-Wire, Analog, and Pulse-timing), design a multi-state real-time municipal dashboard, write streaming parsers for serial GPS streams without using arrays or buffers, manage SPI bus sharing among multiple peripherals, and establish robust fail-safe states for city node systems.

## What You Will Build
An autonomous Smart City municipal node. The system runs on a RISC-V VEGA ARIES v3 board and implements:
1. **Environmental Monitoring:** DHT22 temperature/humidity tracking and MQ-2 smoke/air quality sampling.
2. **Infrastructure/Traffic Alert:** HC-SR04 ultrasonic sensor measures vehicle/pedestrian distance.
3. **Public Transit RFID Check-In:** MFRC522 scans public fobs.
4. **GPS Tracking:** Reads Neo-6M NMEA telemetry stream.
5. **Data Logger:** Appends structured telemetry to `citylog.csv` on the SD card every 5 seconds.
6. **Information LCD Dashboard:** A multi-page rolling HUD displays real-time parameters.
7. **Emergency Siren:** Activates Warning LED (GPIO 15) and Buzzer (GPIO 14) on proximity breaches (<20cm) or smoke limits (>400).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Micro SD Card Module | `sd_card` | Yes | Yes |
| MFRC522 RFID Reader | `rfid_reader` | Yes | Yes |
| RFID Public Transit Fob | `rfid_tag` | Yes | Yes |
| HC-SR04 Ultrasonic Sensor | `ultrasonic` | Yes | Yes |
| DHT22 Temp & Humidity Sensor | `dht22` | Yes | Yes |
| MQ-2 Smoke/Gas Sensor | `mq2` | Yes | Yes |
| Neo-6M GPS Module (UART) | `gps` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| MFRC522 | SDA (CS) | GPIO 10 | Blue | SPI RFID Chip Select |
| MFRC522 | RST | GPIO 9 | Purple | RFID Reset line |
| SD Card Module | CS | GPIO 8 | Orange | SPI SD Card Chip Select |
| SPI Bus (Shared) | MOSI | GP11 (SPI MOSI) | Yellow | Shared SPI Master Out |
| SPI Bus (Shared) | MISO | GP12 (SPI MISO) | Green | Shared SPI Master In |
| SPI Bus (Shared) | SCK | GP13 (SPI SCK) | White | Shared SPI Clock |
| DHT22 Sensor | DATA | GPIO 2 | Yellow | One-wire climate pin (freed from GP12 MISO conflict) |
| MQ-2 Sensor | AO (Analog Out)| ADC2 (GP28) | Orange | Analog gas detection |
| HC-SR04 | Trigger | GPIO 3 | Blue | Ultrasonic trigger |
| HC-SR04 | Echo | GPIO 7 | Grey | Ultrasonic echo response |
| Neo-6M GPS | TX | GPIO 0 (RX1) | Yellow | GPS transmit to ARIES RX1 |
| Neo-6M GPS | RX | GPIO 1 (TX1) | Green | GPS receive from ARIES TX1 |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Active Buzzer | + | GPIO 14 | Orange | Municipal alert siren |
| Warning LED | Anode | GPIO 15 | Orange | Hazard indicator light |

> **Wiring tip:** When sharing the SPI bus between the SD Card and RFID reader, both VCC pins must be wired to their appropriate rails (SD card = 5V, RFID = 3V3). The SCK, MISO, and MOSI lines are wired in parallel. Ensure GPIO 10 (RFID CS) and GPIO 8 (SD CS) are kept separate.

## System Architecture Description
The capstone node represents an integrated smart city municipal hub. Telemetry data flow is visualised below:

```
                      +-----------------------------+
                      |    VEGA ARIES v3 Board      |
                      |  (RISC-V 32-bit Core, 120MHz)|
                      +--------------+--------------+
                                     |
    +-------------------+------------+------------+-------------------+
    |                   |            |            |                   |
+---v---+           +---v---+    +---v---+    +---v---+           +---v---+
|  SPI  |           |  I2C  |    | UART  |    |1-Wire |           |Analog |
|  Bus  |           |  Bus  |    | Port  |    |  Bus  |           |  ADC  |
+---+---+           +---+---+    +---+---+    +---+---+           +---+---+
    |                   |            |            |                   |
    +--> SD Card (GP8)  |            |            |                   |
    |                   +--> LCD     +--> GPS     +--> DHT22 (GP2)    +--> MQ-2 (GP28)
    +--> RFID (GP10)        (GP16/17)    (GP0/1)
```

1. **Hardware Communication Topology:**
   * **SPI Bus (SPI0):** Runs at 4 MHz. Coordinates data transfers with the SD Card and RFID reader. Hardware chip select lines manage device isolation.
   * **I2C Bus (I2C0):** Runs at 100 kHz. Displays real-time parameters on the character display.
   * **UART Port (UART1):** Dedicated hardware serial receiver parsing GPS NMEA-0183 standard messages ($GPRMC sentences).
   * **One-Wire Digital Pin:** Employs precise microsecond transitions on GPIO 2 to read 40-bit climate frames.
   * **Analog Input:** Captures continuous voltage ratings on ADC2 (GP28) for chemical concentrations.
   * **Pulse-Width Timing:** Employs high-resolution counter limits on GPIO 7 to capture ultrasonic echo returns.
2. **Electrical Routing & Power Constraints:**
   * **Transient Currents:** High actuators (Buzzer, Relays, HC-SR04 pulses) and the MQ-2 internal heater draw peak currents up to 300mA. The ARIES onboard LDO regulator cannot supply this safely under full load. In hardware configurations, isolate the 5V power bus using bypass capacitors (100uF) at the power terminals.
   * **Signal Integrity:** SPI high-frequency lines are susceptible to cross-talk. Ground wires must be routed adjacent to SPI lines to shield signals from EMI.

## Code
```cpp
// 200 - RISC-V Superproject: Smart City Node
#include <SPI.h>
#include <SD.h>
#include <Wire.h>
#include <DHT.h>
#include <MFRC522.h>
#include <LiquidCrystal_I2C.h>

// SPI Chip Select pins
const int RFID_CS = 10;
const int RFID_RST = 9;
const int SD_CS = 8;

// Sensors & Actuators
const int DHT_PIN = 2;
const int MQ2_PIN = 28;     // ADC2
const int TRIG_PIN = 3;
const int ECHO_PIN = 7;
const int BUZZER_PIN = 14;
const int WARN_LED = 15;

#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);
MFRC522 rfid(RFID_CS, RFID_RST);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// Telemetry state variables (scalar - NO array buffers allowed)
float temp = 0.0;
float hum = 0.0;
int gasVal = 0;
long echoDuration = 0;
int distanceCm = 0;
int rfidScanned = 0;
int sdStatus = 0; // 0 = Idle, 1 = Ready, 2 = Fault

// Scalar variables to store scanned RFID Card bytes
byte lastCard0 = 0x00;
byte lastCard1 = 0x00;
byte lastCard2 = 0x00;
byte lastCard3 = 0x00;

// GPS parsing state machine (Streaming byte parser - NO buffer arrays)
int gpsState = 0;      // 0 = Search '$', 1='G', 2='P', 3='R', 4='M', 5='C', 6=Parse Fields
int gpsFieldIndex = 0;
int gpsCharIndex = 0;

// Simplified scalar coordinate values parsed on the fly
float gpsLatitude = 0.0;
float gpsLongitude = 0.0;
int gpsValid = 0;

// System timers
int dashboardPage = 0; // 0 = Env, 1 = Safety, 2 = GPS
int displayTimer = 0;
int logTimer = 0;
int buzzerPulse = 0;

void setup() {
  Serial.begin(115200);   // Primary serial logs
  Serial1.begin(9600);    // GPS Neo-6M on RX1/TX1 (GP0/GP1)
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("SMART CITY NODE");
  lcd.setCursor(0, 1);
  lcd.print("CORE INIT...");

  dht.begin();

  // Initialize SPI bus and MFRC522
  SPI.begin();
  rfid.PCD_Init();

  // Initialize SD Card
  if (SD.begin(SD_CS)) {
    sdStatus = 1;
    File logFile = SD.open("citylog.csv", FILE_WRITE);
    if (logFile) {
      if (logFile.size() == 0) {
        logFile.println("Temp_C,Hum_Pct,Gas_Raw,Dist_cm,RFID_Scanned,Lat,Lon");
      }
      logFile.close();
    }
  } else {
    sdStatus = 2;
  }

  // Actuator pin configurations
  pinMode(TRIG_PIN, OUTPUT);
  pinMode(ECHO_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARN_LED, OUTPUT);

  digitalWrite(TRIG_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(WARN_LED, LOW);

  delay(1500);
  lcd.clear();
  Serial.println("Smart City Node Online. System Running.");
}

void loop() {
  // --- 1. STREAMING GPS SERIAL PARSER ---
  // Read Serial1 byte-by-byte without loops or string buffer arrays
  if (Serial1.available() > 0) {
    char c = Serial1.read();
    
    // NMEA Sentence detection state machine
    if (c == '$') {
      gpsState = 1;
      gpsFieldIndex = 0;
      gpsCharIndex = 0;
    } else if (gpsState == 1 && c == 'G') {
      gpsState = 2;
    } else if (gpsState == 2 && c == 'P') {
      gpsState = 3;
    } else if (gpsState == 3 && c == 'R') {
      gpsState = 4;
    } else if (gpsState == 4 && c == 'M') {
      gpsState = 5;
    } else if (gpsState == 5 && c == 'C') {
      gpsState = 6; // Valid $GPRMC header found
    } else if (gpsState == 6) {
      // Parse CSV fields in the GPRMC sentence
      if (c == ',') {
        gpsFieldIndex++;
        gpsCharIndex = 0;
      } else {
        gpsCharIndex++;
        // Field 2 = Active status (A = Valid, V = Void)
        if (gpsFieldIndex == 2 && gpsCharIndex == 1) {
          gpsValid = (c == 'A') ? 1 : 0;
        }
        // Field 3 = Latitude (Parse simplified value from stream)
        if (gpsFieldIndex == 3) {
          if (gpsCharIndex == 1) gpsLatitude = (c - '0') * 10.0;
          if (gpsCharIndex == 2) gpsLatitude += (c - '0');
          if (gpsCharIndex == 3) gpsLatitude += (c - '0') * 0.1;
          if (gpsCharIndex == 4) gpsLatitude += (c - '0') * 0.01;
        }
        // Field 5 = Longitude (Parse simplified value from stream)
        if (gpsFieldIndex == 5) {
          if (gpsCharIndex == 1) gpsLongitude = (c - '0') * 100.0;
          if (gpsCharIndex == 2) gpsLongitude += (c - '0') * 10.0;
          if (gpsCharIndex == 3) gpsLongitude += (c - '0');
          if (gpsCharIndex == 4) gpsLongitude += (c - '0') * 0.1;
          if (gpsCharIndex == 5) gpsLongitude += (c - '0') * 0.01;
        }
      }
    }
  }

  // --- 2. SENSOR SAMPLING ---
  // Read DHT22
  float rTemp = dht.readTemperature();
  float rHum = dht.readHumidity();
  if (!isnan(rTemp)) temp = rTemp;
  if (!isnan(rHum)) hum = rHum;

  // Read MQ-2 Gas
  gasVal = analogRead(MQ2_PIN);

  // Read HC-SR04 Proximity
  digitalWrite(TRIG_PIN, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG_PIN, LOW);
  echoDuration = pulseIn(ECHO_PIN, HIGH, 30000); // 30ms timeout
  if (echoDuration > 0) {
    distanceCm = echoDuration * 0.034 / 2;
  } else {
    distanceCm = 400; // Out of range baseline
  }

  // --- 3. RFID TRANSIT SCANNER ---
  if (rfid.PICC_IsNewCardPresent() && rfid.PICC_ReadCardSerial()) {
    rfidScanned = 1;
    lastCard0 = rfid.uid.uidByte[0];
    lastCard1 = rfid.uid.uidByte[1];
    lastCard2 = rfid.uid.uidByte[2];
    lastCard3 = rfid.uid.uidByte[3];
    
    // Scan alert
    digitalWrite(BUZZER_PIN, HIGH);
    delay(100);
    digitalWrite(BUZZER_PIN, LOW);

    rfid.PICC_HaltA();
    rfid.PCD_StopCrypto1();
  }

  // --- 4. SAFETY SIREN & ALERT MECHANISM ---
  // Trigger alarm if gas exceeds threshold or pedestrian is too close
  if (gasVal > 400 || distanceCm < 20) {
    digitalWrite(WARN_LED, HIGH);
    buzzerPulse++;
    if (buzzerPulse % 2 == 0) {
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  } else {
    digitalWrite(WARN_LED, LOW);
    digitalWrite(BUZZER_PIN, LOW);
  }

  // --- 5. LOGGING CYCLE (Every 5 seconds = 25 ticks * 200 ms) ---
  logTimer++;
  if (logTimer >= 25) {
    logTimer = 0;

    // Log to SD card if working
    if (sdStatus == 1) {
      File logFile = SD.open("citylog.csv", FILE_WRITE);
      if (logFile) {
        logFile.print(temp, 1);     logFile.print(",");
        logFile.print(hum, 1);      logFile.print(",");
        logFile.print(gasVal);      logFile.print(",");
        logFile.print(distanceCm);  logFile.print(",");
        logFile.print(rfidScanned); logFile.print(",");
        logFile.print(gpsLatitude, 2);  logFile.print(",");
        logFile.println(gpsLongitude, 2);
        logFile.close();
        Serial.println(">> Smart City Log saved to SD card.");
      }
    }

    // Output status stream to primary Serial
    Serial.print("[CITY TELEMETRY] Temp: "); Serial.print(temp, 1);
    Serial.print("C | Gas: "); Serial.print(gasVal);
    Serial.print(" | Dist: "); Serial.print(distanceCm);
    Serial.print("cm | Transit Fob: ");
    if (rfidScanned == 1) {
      Serial.print(lastCard0, HEX); Serial.print(lastCard1, HEX);
    } else {
      Serial.print("None");
    }
    Serial.print(" | GPS: ");
    if (gpsValid == 1) {
      Serial.print(gpsLatitude, 2); Serial.print("N, "); Serial.print(gpsLongitude, 2); Serial.println("E");
    } else {
      Serial.println("Searching...");
    }
  }

  // --- 6. DASHBOARD PAGE TOGGLER (Every 2 seconds = 10 ticks * 200 ms) ---
  displayTimer++;
  if (displayTimer >= 10) {
    displayTimer = 0;
    dashboardPage = (dashboardPage + 1) % 3;
    lcd.clear();
  }

  // Render LCD interface page
  if (dashboardPage == 0) {
    lcd.setCursor(0, 0);
    lcd.print("T:"); lcd.print(temp, 1); lcd.print("C H:"); lcd.print(hum, 0); lcd.print("%  ");
    lcd.setCursor(0, 1);
    lcd.print("Gas (MQ2): "); lcd.print(gasVal); lcd.print(" ");
  } 
  else if (dashboardPage == 1) {
    lcd.setCursor(0, 0);
    lcd.print("Traffic D:"); lcd.print(distanceCm); lcd.print("cm ");
    lcd.setCursor(0, 1);
    lcd.print("Fob: ");
    if (rfidScanned == 1) {
      lcd.print(lastCard0, HEX); lcd.print(lastCard1, HEX); lcd.print(lastCard2, HEX);
    } else {
      lcd.print("No Scan     ");
    }
  } 
  else if (dashboardPage == 2) {
    lcd.setCursor(0, 0);
    lcd.print("GPS: ");
    if (gpsValid == 1) {
      lcd.print(gpsLatitude, 1); lcd.print("N "); lcd.print(gpsLongitude, 1); lcd.print("E ");
    } else {
      lcd.print("No Signal   ");
    }
    lcd.setCursor(0, 1);
    lcd.print("SD Status: ");
    lcd.print(sdStatus == 1 ? "OK  " : "FAIL");
  }

  delay(200); // Main system clock tick
}
```

## What to Click in MbedO
1. Drag **VEGA ARIES v3**, **SD Card Module**, **MFRC522 RFID**, **authorized tag**, **HC-SR04**, **DHT22**, **MQ-2**, **Neo-6M GPS**, **I2C LCD**, **Active Buzzer**, and **Warning LED** onto the canvas.
2. Wire the SPI bus in parallel between the SD Card and RFID reader. Connect SD CS to **GPIO 8** and RFID CS to **GPIO 10**.
3. Wire DHT22 to **GPIO 2**; MQ-2 to **ADC2 (GP28)**; HC-SR04: Trig to **GPIO 3**, Echo to **GPIO 7**.
4. Wire GPS: **TX → GPIO 0 (RX1)**, **RX → GPIO 1 (TX1)**.
5. Wire I2C LCD: **SDA → SDA0 (GP17)**, **SCL → SCL0 (GP16)**.
6. Wire Buzzer to **GPIO 14** and Warning LED to **GPIO 15**.
7. Paste the code into the editor. Select **Interpreted Mode** in the dropdown.
8. Click **Run**.
9. Interact with the widgets: adjust distance slider below 20cm, MQ-2 smoke above 400, or drag the RFID card near the antenna. Watch the rotating LCD pages and Serial outputs.

## Expected Output
Serial Monitor:
```
Smart City Node Online. System Running.
>> Smart City Log saved to SD card.
[CITY TELEMETRY] Temp: 24.5C | Gas: 120 | Dist: 180cm | Transit Fob: None | GPS: Searching...
>> Smart City Log saved to SD card.
[CITY TELEMETRY] Temp: 24.6C | Gas: 122 | Dist: 15cm | Transit Fob: None | GPS: 12.97N, 77.59E
>> Smart City Log saved to SD card.
[CITY TELEMETRY] Temp: 24.5C | Gas: 480 | Dist: 180cm | Transit Fob: 83A2C3 | GPS: 12.97N, 77.59E
```

## Expected Canvas Behavior
* Startup: LCD prints `SMART CITY NODE CORE INIT...`. The Warning LED is OFF.
* Rolling HUD: The LCD changes pages every 2 seconds:
  * Page 1 shows temperature, humidity, and MQ-2 smoke level.
  * Page 2 shows traffic distance and last RFID scan hex bytes.
  * Page 3 displays GPS latitude/longitude coordinates and SD card status.
* Alarm Condition: If the distance slider is pulled below 20cm or smoke slider raised above 400, the Warning LED lights up and the buzzer rings in a pulsing cadence.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `SD.begin(8)` | Initializes the SD card module using CS pin GP8. |
| `rfid.PCD_Init()` | Initializes the MFRC522 RFID reader over the shared SPI bus. |
| `Serial1.read()` | Reads one byte at a time from the GPS serial transmitter on GP0. |
| `gpsState == 6` | Tracks GPRMC sentences to parse parameters on the fly. |
| `pulseIn(7, HIGH, 30000)` | Calculates distance by measuring echo return pulse width in microseconds. |
| `displayTimer >= 10` | Cycles display screens without blocking delay states. |
| `logFile.print(...)` | Appends comma-separated values to the SD file `citylog.csv`. |

## Hardware & Safety Concept
* **Municipal Infrastructure Security:** Smart city nodes monitor public health and safety. Fire hazards (gas leaks) and pedestrian safety (blind spots or proximity collisions) must have local hardware fail-safes. The state machine operates indicators locally without waiting for cloud authorization.
* **SPI Bus Arbitration:** When sharing a SPI bus, only one Chip Select pin is pulled LOW at a time. The SD Card and MFRC522 use separate CS lines. Software ensures a file is closed (`logFile.close()`) before reading the RFID register to prevent data corruption.

## Try This! (Challenges)
1. **Low-power Night State:** Use the GPS time output to enter a low-power mode between 01:00:00 and 05:00:00. Turn off the LCD backlight and reduce sensor polling rates to conserve power.
2. **Speed Trap Monitor:** Connect a second HC-SR04 on GPIO pins. Calculate vehicle speed by measuring the time offset between triggering the first and second sensor, saving speed logs to the SD Card.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| SD Card fails to mount | Pin GP12 (MISO) conflict | Ensure DHT22 is wired to GPIO 2. SPI pins (GP10-GP13) must remain clear. |
| GPS longitude/latitude reads 0 | No sat line-of-sight | Ensure the GPS antenna has a clear outdoor line-of-sight. Verify TX connects to RX1 (GP0). |
| LCD dashboard flickers | Screen cleared too fast | Remove `lcd.clear()` from the main loop; only call clear during page changes. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [77 - HC-SR04 Distance Bar Graph LCD](../intermediate/77-hc-sr04-distance-bar-graph-lcd.md)
- [136 - SD Card Data Logger](../advanced/136-sd-card-data-logger.md)
- [199 - Smart Beehive Monitor](199-smart-beehive-monitor.md)
