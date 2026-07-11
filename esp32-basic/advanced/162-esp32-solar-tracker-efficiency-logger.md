# 162 - ESP32 Solar Tracker Efficiency Logger

Build a complete solar energy logging station that tracks the sun using a dual-axis LDR and servo frame, measures panel voltage and current generation, and logs tracking angles and power efficiency to an SPI SD card.

## Goal
Learn how to combine multi-sensor tracking logic with analog power telemetry (voltage and current), calculate power output, and write datasets to SD cards.

## What You Will Build
A solar tracker is controlled by LDRs (GPIO 34, 35, 32, 33) and servos (GPIO 13, 12). A voltage divider on GPIO 26 and an ACS712 current sensor on GPIO 25 measure solar panel output. An SD card reader is on SPI (CS: 15), and a status LED on GPIO 4. Every 10 seconds, tracking positions and power output are logged to `/solar.csv`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| LDRs (Photoresistors) (4) | `ldr` | Yes | Yes |
| Servos (2) | `servo` | Yes | Yes |
| ACS712 Current Sensor (5A) | `acs712` | Yes | Yes |
| MicroSD Card Shield (SPI) | `sd_card` | Yes | Yes |
| LED (Green) | `led` | Yes | Yes |
| 10 kΩ / 4.7 kΩ Resistors (for divider) | `resistor` | No | Yes |
| 10 kΩ Resistors (4, for LDRs) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Solar Tracker | LDR / Servo Pins | GPIO34 / 35 / 32 / 33 / 13 / 12 | Various | Tracker wiring (Project 161) |
| Voltage Divider | Output | GPIO26 | White | Scaled panel voltage |
| ACS712 Sensor | OUT (Analog) | GPIO25 | Yellow | Panel current generation |
| SD Card Module | CS / SCK / MISO / MOSI | GPIO15 / 18 / 19 / 23 | Purple/Yellow/Green/Blue | SPI interface |
| Green LED | Anode (+) | GPIO4 via 330 Ω | Green | Write confirmation |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** The voltage divider scales the solar panel voltage down to a safe range (0–3.3V) for the ESP32 analog pin. Ensure all grounds are tied to a common rail.

## Code
```cpp
// Solar Tracker Efficiency Logger (SD + ACS712 + Servos + LDRs)
#include <SPI.h>
#include <FS.h>
#include <SD.h>
#include <ESP32Servo.h>

// Pins definitions
const int LDR_TL = 34;
const int LDR_TR = 35;
const int LDR_BL = 32;
const int LDR_BR = 33;

const int SERVO_H = 13;
const int SERVO_V = 12;

const int CURRENT_PIN = 25;
const int VOLTAGE_PIN = 26;
const int LED_PIN = 4;
const int SD_CS = 15;

Servo horizontalServo;
Servo verticalServo;

int hAngle = 90;
int vAngle = 90;

// Calibration factors
const float R_DIVIDER_FACTOR = 4.13; // Scaled for 10k / 3.1k resistor divider (max 13.6V input)
const float ACS_OFFSET = 1.65;       // Zero-current voltage offset (3.3V / 2)
const float ACS_SENSITIVITY = 0.185; // 185 mV/A for 5A ACS712 model

unsigned long lastLogTime = 0;
const unsigned long LOG_INTERVAL_MS = 10000; // Log data every 10 seconds

void writeCSVHeader(fs::FS &fs, const char * path) {
  if (!fs.exists(path)) {
    Serial.println("Creating solar log file...");
    File file = fs.open(path, FILE_WRITE);
    if (file) {
      file.println("Time(s),PanAngle,TiltAngle,Voltage(V),Current(mA),Power(mW)");
      file.close();
    }
  }
}

void logSolarData(fs::FS &fs, const char * path, int pan, int tilt, float volt, float curr, float pwr) {
  File file = fs.open(path, FILE_APPEND);
  if (!file) {
    Serial.println("Failed to write to SD card!");
    return;
  }
  
  String csvRow = String(millis() / 1000) + "," + String(pan) + "," + String(tilt) + "," + 
                  String(volt, 2) + "," + String(curr, 1) + "," + String(pwr, 1);
                  
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

float readPanelVoltage() {
  int raw = analogRead(VOLTAGE_PIN);
  float voltage = raw * (3.3 / 4095.0);
  return voltage * R_DIVIDER_FACTOR;
}

float readPanelCurrentmA() {
  int raw = analogRead(CURRENT_PIN);
  float voltage = raw * (3.3 / 4095.0);
  float currentA = (voltage - ACS_OFFSET) / ACS_SENSITIVITY;
  if (abs(currentA) < 0.03) currentA = 0.0; // Filter noise
  return currentA * 1000.0;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW);
  
  horizontalServo.attach(SERVO_H);
  verticalServo.attach(SERVO_V);
  horizontalServo.write(hAngle);
  verticalServo.write(vAngle);
  
  Serial.println("Mounting SD Card...");
  if (!SD.begin(SD_CS)) {
    Serial.println("SD Card Mount Failed!");
    while(1) {}
  }
  
  writeCSVHeader(SD, "/solar.csv");
  Serial.println("Solar efficiency logger online.");
}

void loop() {
  // 1. Run Active Solar Tracking Control Loop
  int tl = analogRead(LDR_TL);
  int tr = analogRead(LDR_TR);
  int bl = analogRead(LDR_BL);
  int br = analogRead(LDR_BR);
  
  int avgTop    = (tl + tr) / 2;
  int avgBottom = (bl + br) / 2;
  int avgLeft   = (tl + bl) / 2;
  int avgRight  = (tr + br) / 2;
  
  int diffVert = avgTop - avgBottom;
  int diffHoriz = avgLeft - avgRight;
  
  // Slow adjustments to prevent servo jitter
  if (abs(diffVert) > 150) {
    if (avgTop > avgBottom) vAngle = constrain(vAngle - 1, 15, 150);
    else vAngle = constrain(vAngle + 1, 15, 150);
    verticalServo.write(vAngle);
  }
  
  if (abs(diffHoriz) > 150) {
    if (avgLeft > avgRight) hAngle = constrain(hAngle + 1, 10, 170);
    else hAngle = constrain(hAngle - 1, 10, 170);
    horizontalServo.write(hAngle);
  }
  
  // 2. Log Power Telemetry every 10 seconds
  unsigned long now = millis();
  if (now - lastLogTime >= LOG_INTERVAL_MS) {
    float volt = readPanelVoltage();
    float currmA = readPanelCurrentmA();
    
    // Power (mW) = Voltage (V) * Current (mA)
    float power = volt * currmA;
    if (power < 0.0) power = 0.0;
    
    logSolarData(SD, "/solar.csv", hAngle, vAngle, volt, currmA, power);
    
    lastLogTime = now;
  }
  
  delay(30); // 33Hz loop
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, four **LDRs**, two **Servos**, **ACS712**, **SD Card**, and **LED** onto the canvas.
2. Wire tracking inputs, ACS712 to **GPIO25**, voltage input to **GPIO26**, SD CS to **GPIO15**, and LED to **GPIO4**.
3. Paste the code and click **Run**.
4. Slide LDR values to trigger tracking.
5. Adjust current and voltage sliders. Watch the Green LED flash every 10 seconds as it logs tracking angles and power to `/solar.csv`.

## Expected Output
Serial Monitor:
```
Mounting SD Card...
Solar efficiency logger online.
Logged: 10,92,88,12.20,120.4,1468.9
Logged: 20,93,87,12.10,135.0,1633.5
```

`/solar.csv` file contents:
```
Time(s),PanAngle,TiltAngle,Voltage(V),Current(mA),Power(mW)
10,92,88,12.20,120.4,1468.9
20,93,87,12.10,135.0,1633.5
```

## Expected Canvas Behavior
* Adjusting LDR sliders moves the horizontal and vertical servo widgets.
* Every 10 seconds, the Green LED widget flashes.
* Adjusting the voltage and current sliders updates the power values logged to the `/solar.csv` output.

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `readPanelVoltage()` | Measures analog voltage and scales it back up based on the resistor divider factor. |
| `readPanelCurrentmA()` | Measures current from the ACS712 voltage reading and converts it to milliamperes. |
| `volt * currmA` | Computes generated solar power in milliwatts. |

## Hardware & Safety Concept: Solar Power Logging and Efficiency Auditing
Solar power logging allows auditing solar panels in real time. Power generation fluctuates depending on weather conditions (clouds) and the panel's orientation relative to the sun. Logging tracking angles alongside generated power allows analyzing efficiency gains to verify that the energy consumed by the servos is less than the extra energy harvested by tracking the sun.

## Try This! (Challenges)
1. **Interactive Log Dump**: Add a button on GPIO 14 that dumps the entire log file to the Serial Monitor.
2. **Dynamic Calibration Mode**: Write a startup routine that automatically calculates LDR calibration offsets.
3. **Low-power Hibernate**: Disable the tracking servos (using `servo.detach()`) if LDR light levels are extremely low for more than 5 minutes to conserve power.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Servos vibrate and oscillate rapidly | Tolerance setting too low | Increase the tolerance value in the tracking checks |
| Power calculations are negative | ACS712 current sensor offset incorrect | Adjust the `ACS_OFFSET` variable in the code (it should be half of the sensor's supply voltage) |
| SD card fails to mount | Pin mapping conflict | Confirm SD CS pin is wired to GPIO 15, not GPIO 5 |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [161 - ESP32 Solar Tracker Dual Axis Controller](161-esp32-solar-tracker-dual-axis-controller.md)
- [138 - ESP32 SD Card Current/Energy Logger](138-esp32-sd-card-current-energy-logger.md)
- [136 - ESP32 SD Card Data Logger](136-esp32-sd-card-data-logger.md)
