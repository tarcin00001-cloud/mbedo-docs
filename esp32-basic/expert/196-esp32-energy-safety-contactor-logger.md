# 196 - ESP32 Energy Safety Contactor Logger

Build an industrial power monitoring station on the ESP32 that measures load current using an ACS712 sensor, calculates active power (Watts) and energy consumption (Watt-hours), logs stats to an SPI SD card, and trips a contactor relay if current limits are exceeded.

## Goal
Learn how to sample current waveforms, calculate electrical power (Watts) and integrate energy consumption (Wh), manage SD card CSV files, and implement safety trip-outs.

## What You Will Build
An ACS712 current sensor is connected to GPIO 34. A contactor relay is on GPIO 13, a status LED on GPIO 12, and an SD card reader on SPI (CS: 15). The ESP32 measures current and calculates power usage (assuming a 230V load). Every 5 seconds, it appends values to `/energy_log.csv` on the SD card and flashes the LED. If current exceeds 2.0A, the relay opens instantly.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor (5A) | `acs712` | Yes | Yes |
| Relay Module (Contactor) | `relay` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT (Analog) | GPIO34 | Yellow | Load current input |
| Relay Module | IN (Signal) | GPIO13 | Orange | Load contactor switch |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Log confirmation LED |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The SD card module Chip Select connects to GPIO 15. Power the contactor relay from the 5V Vin rail.

## Code
```cpp
// Energy Safety Contactor Logger (ACS712 + Relay + SD Card)
#include <SPI.h>
#include <FS.h>
#include <SD.h>

const int CURRENT_PIN = 34;
const int RELAY_PIN = 13;
const int LED_PIN = 12;
const int SD_CS = 15;

// Calibration parameters
const float ACS_OFFSET = 1.65;         // Zero-current voltage offset (3.3V / 2)
const float ACS_SENSITIVITY = 0.185;   // 185 mV/A for 5A ACS712 sensor
const float SYSTEM_VOLTAGE = 230.0;    // Mains reference voltage (for power calculations)
const float CURRENT_LIMIT_MA = 2000.0; // Overcurrent trip threshold (2.0 A)

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log data every 5 seconds

float accumulatedWh = 0.0;
bool systemActive = true;

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(s),Current(mA),Power(W),Energy(Wh),Status");
      file.close();
    }
  }
}

void logPowerData(fs::FS &fs, const char * path, float currentmA, float powerW, float energyWh, const char* status) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to write to SD card!");
    return;
  }
  
  String csvRow = String(millis() / 1000) + "," + String(currentmA, 1) + "," + 
                  String(powerW, 2) + "," + String(energyWh, 4) + "," + status;
                  
  if (file.println(csvRow)) {
    Serial.print("Logged: "); Serial.println(csvRow);
    digitalWrite(LED_PIN, HIGH);
    delay(100);
    digitalWrite(LED_PIN, LOW);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

float readCurrentmA() {
  long sum = 0;
  const int NUM_SAMPLES = 100;
  for (int i = 0; i < NUM_SAMPLES; i++) {
    sum += analogRead(CURRENT_PIN);
    delayMicroseconds(50);
  }
  float avgRaw = (float)sum / NUM_SAMPLES;
  float voltage = avgRaw * (3.3 / 4095.0);
  
  float currentA = (voltage - ACS_OFFSET) / ACS_SENSITIVITY;
  if (abs(currentA) < 0.03) currentA = 0.0; // Filter noise
  return currentA * 1000.0;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(LED_PIN, OUTPUT);
  
  digitalWrite(RELAY_PIN, HIGH); // Close contactor on boot (power ON)
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {}
  }
  
  writeCSVHeader(SD, "/energy_log.csv");
  Serial.println("Energy safety logger active.");
}

void loop() {
  if (!systemActive) return;
  
  float currentmA = readCurrentmA();
  unsigned long now = millis();
  
  // 1. Safety Overcurrent Check
  if (abs(currentmA) >= CURRENT_LIMIT_MA) {
    // Open contacts instantly (power OFF)
    digitalWrite(RELAY_PIN, LOW);
    systemActive = false;
    
    Serial.println("!!! OVERCURRENT TRIP: OPENING CONTACTS !!!");
    logPowerData(SD, "/energy_log.csv", currentmA, 0.0, accumulatedWh, "TRIPPED");
    return;
  }
  
  // 2. Periodic Power Log (every 5 seconds)
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    float dtHours = (float)LOG_INTERVAL_MS / 3600000.0;
    
    // Power (W) = Voltage (V) * Current (A)
    float currentA = currentmA / 1000.0;
    float powerW = SYSTEM_VOLTAGE * abs(currentA);
    
    // Integrate: Wh = Watts * hours
    accumulatedWh += powerW * dtHours;
    
    logPowerData(SD, "/energy_log.csv", currentmA, powerW, accumulatedWh, "ACTIVE");
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ACS712**, **Relay**, **SD Card**, and **LED** onto the canvas.
2. Wire ACS712 to **GPIO34**, Relay to **GPIO13**, SD CS to **GPIO15**, and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Slide the ACS712 current widget to 500 mA. Watch the Green LED flash every 5 seconds as it logs to `/energy_log.csv`.
5. Slide the current widget above 2.0A. Watch the relay turn OFF instantly and log `TRIPPED` to the SD card.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Energy safety logger active.
Logged: 5,420.0,96.60,0.1342,ACTIVE
Logged: 10,410.0,94.30,0.2652,ACTIVE
!!! OVERCURRENT TRIP: OPENING CONTACTS !!!
```

`/energy_log.csv` file contents:
```
Time(s),Current(mA),Power(W),Energy(Wh),Status
5,420.0,96.60,0.1342,ACTIVE
10,410.0,94.30,0.2652,ACTIVE
12,2120.0,0.00,0.2652,TRIPPED
```

## Expected Canvas Behavior
* At boot, the relay widget turns ON (green).
* Every 5 seconds, the Green LED widget flashes.
* Adjusting the current slider changes the power and energy values logged to the `/energy_log.csv` output.
* Sliding the current slider above 2000 mA turns the relay widget OFF (grey).

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `readCurrentmA()` | Takes an average of 100 samples to filter out transient startup spikes. |
| `powerW = SYSTEM_VOLTAGE * abs(currentA)` | Calculates power usage assuming a 230V reference voltage. |
| `accumulatedWh += powerW * dtHours` | Integrates power usage over time to calculate cumulative consumption. |

## Hardware & Safety Concept: Industrial Power Auditing and Safety Trip-outs
Industrial electrical panels monitor current and power usage to audit efficiency and protect hardware. If a machine fails, it can draw a short-circuit current that generates high heat, melting wires or starting a fire. Integrating an **ACS712 Hall effect sensor** allows measuring current non-invasively, logging usage statistics, and tripping a contactor relay instantly if the current exceeds safe thresholds, protecting hardware.

## Try This! (Challenges)
1. **Interactive Reset button**: Add a button on GPIO 4 to reset the tripped state and re-close the relay.
2. **Dynamic Voltage sensor**: Integrate an AC voltage sensor (ZMPT101B) to measure actual line voltage rather than assuming a static 230V.
3. **Weekly energy target alert**: Flash an LED warning if the total energy usage exceeds a target limit (e.g. 50 Wh).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads 0.0 even under load | Sensor output floating | Verify the ACS712 analog output pin is connected to GPIO 34 |
| System trips immediately on boot | ACS712 offset calibration incorrect | Verify that `ACS_OFFSET` matches half of the supply voltage |
| SD card fails to mount | Pin mapping conflict | Confirm SD CS pin is wired to GPIO 15, not GPIO 5 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
- [167 - ESP32 Current Overload Contactor Breaker](167-esp32-current-overload-contactor-breaker.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
