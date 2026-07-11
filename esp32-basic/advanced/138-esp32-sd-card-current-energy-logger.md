# 138 - ESP32 SD Card Current/Energy Logger

Build an electrical energy logging station that measures current using an ACS712 Hall effect sensor, integrates power consumption over time to calculate total energy (in Watt-hours), and logs data to an SPI SD card.

## Goal
Learn how to sample and calibrate Hall effect current sensors, calculate electrical power parameters, perform numerical integration over time, and write datasets to SD cards.

## What You Will Build
An ACS712 current sensor is connected to GPIO 34. An SD card reader is connected via SPI (CS: 5). A status LED is on GPIO 12. Every 5 seconds, the ESP32 samples current, calculates live power (assuming 12V DC system voltage), integrates this to calculate accumulated Watt-hours (Wh), and logs the values to `/energy.csv`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| ACS712 Current Sensor (5A Model) | `acs712` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 330 Ω Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| ACS712 Sensor | OUT (Analog) | GPIO34 | Yellow | Analog current input |
| ACS712 Sensor | VCC / GND | 5V / GND | Red / Black | Sensor power |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO5 / 18 / 19 / 23 | Orange/Yellow/Green/Blue | SPI interface |
| SD Card Module | VCC / GND | 5V / GND | Red / Black | Power rails |
| Green LED | Anode (+) | GPIO12 via 330 Ω | Green | Status indicator |
| Green LED | Cathode (−) | GND | Black | Ground reference |

> **Wiring tip:** The ACS712 requires 5V VCC for stable operation. Since its analog output can reach 5V, use a voltage divider (e.g. 10 kΩ / 20 kΩ) or a 3.3V clamp to scale the output down to 3.3V before connecting to GPIO 34 to protect the ESP32 analog pin.

## Code
```cpp
// SD Card Current/Energy Logger (ACS712 -> SD)
#include <FS.h>
#include <SD.h>
#include <SPI.h>

const int CS_PIN = 5;
const int CURRENT_PIN = 34;
const int LED_PIN = 12;

// ACS712 Calibration parameters
const float V_OFFSET = 1.65;      // Zero-current voltage offset (3.3V / 2)
const float SENSITIVITY = 0.185;  // 185 mV/A for 5A model

// Simulated System Voltage (e.g. 12V DC Battery System)
const float SYSTEM_VOLTAGE_V = 12.0;

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 5000; // Log every 5 seconds

float totalEnergyWh = 0.0; // Accumulated energy in Watt-hours

float readCurrent() {
  int raw = analogRead(CURRENT_PIN);
  float voltage = raw * (3.3 / 4095.0);
  
  // Calculate current: (Vout - Voffset) / Sensitivity
  float current = (voltage - V_OFFSET) / SENSITIVITY;
  
  // Apply deadband noise filter (ignore currents below 0.05A)
  if (abs(current) < 0.05) current = 0.0;
  return current;
}

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(ms),Current(A),Power(W),Energy(Wh)");
      file.close();
    }
  }
}

void appendToCSV(fs::FS &fs, const char * path, String dataRow) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Error opening CSV file!");
    return;
  }
  
  if (file.println(dataRow)) {
    Serial.print("Logged: "); Serial.println(dataRow);
    digitalWrite(LED_PIN, HIGH);
    delay(100);
    digitalWrite(LED_PIN, LOW);
  } else {
    Serial.println("Write failed!");
  }
  file.close();
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(CS_PIN)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {
      digitalWrite(LED_PIN, HIGH); delay(200); digitalWrite(LED_PIN, LOW); delay(200);
    }
  }
  
  writeCSVHeader(SD, "/energy.csv");
  Serial.println("Energy logger active.");
}

void loop() {
  unsigned long now = millis();
  
  // Log on scheduled intervals (non-blocking)
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    float currentA = readCurrent();
    
    // Power (Watts) = Voltage * Current
    float powerW = SYSTEM_VOLTAGE_V * currentA;
    
    // Integrate energy: Wh = Power (W) * Time (hours)
    // Time interval in hours = 5 seconds / 3600 seconds
    float hours = (float)LOG_INTERVAL_MS / 3600000.0;
    totalEnergyWh += powerW * hours;
    
    // Format CSV row
    String csvRow = String(now) + "," + String(currentA, 3) + "," + String(powerW, 2) + "," + String(totalEnergyWh, 5);
    
    appendToCSV(SD, "/energy.csv", csvRow);
    
    lastLogTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **ACS712**, **SD Card**, and **LED** onto the canvas.
2. Wire ACS712 OUT to **GPIO34**, SD CS to **GPIO5**, and LED to **GPIO12**.
3. Paste the code and click **Run**.
4. Adjust the current slider on the ACS712 widget. Watch the energy and current values accumulate in `/energy.csv`.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Energy logger active.
Logged: 5000,0.450,5.40,0.00750
Logged: 10000,1.200,14.40,0.02750
```

CSV file contents (`/energy.csv`):
```
Time(ms),Current(A),Power(W),Energy(Wh)
5000,0.450,5.40,0.00750
10000,1.200,14.40,0.02750
```

## Expected Canvas Behavior
* The SD card mounts successfully.
* Every 5 seconds, the green LED widget flashes.
* Adjusting the current slider on the simulated ACS712 changes the current, power, and accumulated energy logged in the `/energy.csv` output.

## Code Walkthrough
| Line | Check / Math |
| --- | --- |
| `voltage - V_OFFSET` | Subtracts the zero-current center reference voltage. |
| `SYSTEM_VOLTAGE_V * currentA` | Calculates instantaneous power consumption in Watts. |
| `totalEnergyWh += powerW * hours` | Integrates power over time to calculate cumulative Watt-hours. |

## Hardware & Safety Concept: Current Shunt and Hall Effect Safety
The ACS712 uses a Hall effect sensor to measure current. The circuit path under test runs through an internal copper path, generating a magnetic field that the Hall sensor detects and converts to a voltage. This design provides **galvanic isolation** (up to 2.1 kV) between the high-current load side and the low-voltage ESP32 logic side. This prevents high-voltage spikes from traveling back to damage the microcontroller.

## Try This! (Challenges)
1. **Interactive Reset button**: Add a button on GPIO 15 that clears the accumulated Watt-hours back to 0.0.
2. **Current Overload Protection**: Add a relay on GPIO 13. If the measured current exceeds 4.0A for more than 2 seconds, trigger the relay to cut power to the load.
3. **Ah Accumulator mode**: Log Ampere-hours (Ah) instead of Watt-hours (Wh) for battery capacity monitoring.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Current reads ~0.0 even under load | Sensor analog pin float | Confirm ACS712 VCC is powered by 5V and output is wired to GPIO 34 |
| Readings jump up and down erratically | High-frequency AC noise | Add a 0.1 µF capacitor across the ACS712 filter pins |
| Write fails | SD card full | Verify remaining space on the SD card |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
- [137 - ESP32 SD Card Temperature Logger](137-esp32-sd-card-temperature-logger.md)
- [31 - ESP32 Potentiometer ADC Read](../beginner/31-esp32-potentiometer-adc-read.md)
