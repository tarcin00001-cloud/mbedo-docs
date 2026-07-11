# 159 - ESP32 Automatic Battery Capacity Tester

Build a smart battery capacity tester (discharge analyzer) that measures battery cell voltage, switches a load resistor relay to discharge the cell, integrates discharge current over time to calculate capacity in mAh, logs progress to an SD card, and cuts off discharge at a safety threshold.

## Goal
Learn how to implement battery discharge loops, integrate current over time to compute capacity (milliampere-hours, mAh), enforce low-voltage cutoff safety limits, and write datasets to SD cards.

## What You Will Build
A battery cell is connected to GPIO 34 through a 10 kΩ / 10 kΩ voltage divider. A relay on GPIO 13 switches a 10 Ω power load resistor. An SD card reader is on SPI (CS: 15), and a buzzer on GPIO 4. The ESP32 discharges the battery at a constant rate, integrating current, and logging the capacity to `/battery_log.csv` every 10 seconds. Once the cell voltage drops below 3.0V (cutoff), the relay shuts off.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Battery (e.g. 18650 Li-ion Cell) | `battery` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| 10 Ω Power Resistor (10W) | `resistor` | No | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 10 kΩ Resistors (2) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Battery (+) | Output | Divider top | Red | Battery positive line |
| Divider Node | Junction | GPIO34 | Yellow | Scaled analog voltage |
| 10 kΩ Resistor 1 | Leg 1 / Leg 2 | Battery (+) / GPIO34 | White | Divider top leg |
| 10 kΩ Resistor 2 | Leg 1 / Leg 2 | GPIO34 / GND | White | Divider bottom leg |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load resistor switch |
| Relay Module | COM / NO | Battery (+) / Load Resistor | Red | Load circuit contacts |
| Load Resistor | Terminals | Relay NO / GND | Black | Discharge dump load |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Active Buzzer | VCC (+) | GPIO4 | Orange | Test complete alarm |
| Common ground | GND / Battery (−) | GND | Black | Shared reference |

> **Wiring tip:** The 10 kΩ / 10 kΩ voltage divider splits the battery voltage in half, allowing the ESP32 to safely measure batteries up to 6.6V. Choose a high-power resistor (e.g. 10W) for the 10 Ω load to handle heat generation during discharge.

## Code
```cpp
// Automatic Battery Capacity Tester (Discharge Logger)
#include <SPI.h>
#include <FS.h>
#include <SD.h>

const int VOLTAGE_PIN = 34;
const int RELAY_PIN = 13;
const int BUZZER_PIN = 4;
const int SD_CS = 15;

// Calibration parameters
const float R_LOAD = 10.0;          // Load resistor resistance in Ohms
const float CUTOFF_VOLTAGE = 3.0;   // Safe low-voltage cutoff limit for Li-ion
const float RES_DIVIDER_FACTOR = 2.0; // 10k/10k divider splits voltage in half

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log every 10 seconds

float accumulatedmAh = 0.0;
bool testActive = true;

float getBatteryVoltage() {
  int raw = analogRead(VOLTAGE_PIN);
  float pinVoltage = raw * (3.3 / 4095.0);
  return pinVoltage * RES_DIVIDER_FACTOR;
}

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(s),Voltage(V),Current(mA),Capacity(mAh)");
      file.close();
    }
  }
}

void logDischarge(fs::FS &fs, const char * path, float voltage, float currentmA, float capacitymAh) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to open log file!");
    return;
  }
  
  // Format CSV row: Time (seconds), Voltage, Current, Capacity
  String csvRow = String(millis() / 1000) + "," + String(voltage, 2) + "," + 
                  String(currentmA, 1) + "," + String(capacitymAh, 2);
                  
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, LOW); // Start with load disconnected
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {}
  }
  
  writeCSVHeader(SD, "/battery_log.csv");
  
  // Perform startup safety check
  float startVoltage = getBatteryVoltage();
  Serial.print("Startup Battery Voltage: "); Serial.print(startVoltage); Serial.println(" V");
  
  if (startVoltage < CUTOFF_VOLTAGE) {
    Serial.println("Battery already below safety cutoff. Test aborted.");
    testActive = false;
  } else {
    Serial.println("Starting discharge test. Connecting load relay...");
    digitalWrite(RELAY_PIN, HIGH); // Connect load resistor
    lastLogTime = millis();
  }
}

void loop() {
  if (!testActive) return;
  
  float battVoltage = getBatteryVoltage();
  unsigned long now = millis();
  
  // 1. Safety Cutoff Check
  if (battVoltage < CUTOFF_VOLTAGE) {
    // Cut off load instantly to protect battery
    digitalWrite(RELAY_PIN, LOW); 
    testActive = false;
    
    Serial.println("!!! CUTOFF REACHED: Battery Discharged !!!");
    Serial.print("Final Capacity: "); Serial.print(accumulatedmAh); Serial.println(" mAh");
    
    // Log final state
    logDischarge(SD, "/battery_log.csv", battVoltage, 0.0, accumulatedmAh);
    
    // Play test-complete alarm chime
    for (int i = 0; i < 3; i++) {
      digitalWrite(BUZZER_PIN, HIGH); delay(200);
      digitalWrite(BUZZER_PIN, LOW);  delay(100);
    }
    return;
  }
  
  // 2. Periodic Capacity Integration & Logging
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    float dtHours = (float)LOG_INTERVAL_MS / 3600000.0; // Interval in hours
    
    // Ohm's Law: I = V / R
    float currentA = battVoltage / R_LOAD;
    float currentmA = currentA * 1000.0;
    
    // Integrate: mAh = mA * hours
    accumulatedmAh += currentmA * dtHours;
    
    logDischarge(SD, "/battery_log.csv", battVoltage, currentmA, accumulatedmAh);
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Battery**, **Relay**, **SD Card**, and **Buzzer** onto the canvas.
2. Wire Battery output to **GPIO34**, Relay to **GPIO13**, SD CS to **GPIO15**, and Buzzer to **GPIO4**.
3. Paste the code and click **Run**.
4. Adjust the battery slider to 4.2V. Watch the relay turn ON and log current/capacity.
5. Drag the battery slider down to 2.9V. Watch the relay turn OFF instantly and the buzzer beep.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Startup Battery Voltage: 4.15 V
Starting discharge test. Connecting load relay...
Logged: 10,4.10,410.0,1.14
Logged: 20,4.05,405.0,2.26
!!! CUTOFF REACHED: Battery Discharged !!!
Final Capacity: 3.14 mAh
```

`/battery_log.csv` file contents:
```
Time(s),Voltage(V),Current(mA),Capacity(mAh)
10,4.10,410.0,1.14
20,4.05,405.0,2.26
23,3.00,0.0,2.34
```

## Expected Canvas Behavior
* At boot, the relay widget turns ON (green) if the battery slider is set to > 3.0V.
* Every 10 seconds, values are appended to the CSV log.
* If you drag the battery slider below 3.0V, the relay widget turns OFF (grey) and the buzzer widget pulses.

## Code Walkthrough
| Line | Math / Check |
| --- | --- |
| `pinVoltage * RES_DIVIDER_FACTOR` | Scales measured voltage back up to absolute battery voltage levels. |
| `battVoltage < CUTOFF_VOLTAGE` | Monitors voltage drops to prevent battery damage from over-discharge. |
| `currentmA * dtHours` | Integrates current over the elapsed hours to calculate accumulated capacity in mAh. |

## Hardware & Safety Concept: Lithium-Ion Battery Over-discharge Risks
Lithium-ion battery cells (like 18650 cells) have a nominal voltage of 3.7V, charging to 4.2V. If they are discharged below **2.5V to 3.0V**, the internal chemical structure decomposes, permanently degrading battery capacity or creating copper shunts that cause internal short circuits during recharge (creating fire hazards). Closed-loop battery analyzers must include **low-voltage cutoff loops** that physically disconnect the battery using a relay or MOSFET once the cutoff limit is reached.

## Try This! (Challenges)
1. **Interactive OLED progress display**: Add an OLED HUD (Project 60) showing live cell voltage, current, and capacity in real time.
2. **Current Shunt upgrade**: Wire an ACS712 current sensor (Project 138) to measure actual discharge current rather than calculating it using Ohm's Law.
3. **Thermal cutoff safety**: Add a thermistor (Project 105) to measure battery temperature, shutting down the relay if it exceeds 50 °C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Battery voltage reads static 0V | Resistor divider miswired | Verify voltage divider resistors are connected to battery (+), GPIO 34, and GND |
| Test aborts immediately at start | Battery already depleted | Recharge the cell above 3.5V before starting the discharge test |
| Capacity values are static | Time variables scaling issue | Ensure `dtHours` is calculated using floating-point math |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
- [105 - ESP32 Thermostat Controller](105-esp32-thermostat-controller.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
