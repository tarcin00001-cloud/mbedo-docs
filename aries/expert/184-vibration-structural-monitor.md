# 184 - Vibration Structural Health Monitor

Build a structural health monitoring node that samples vibration signals from three analog shock/vibration sensors, performs peak-to-peak calculation, logs events to an SD card, and displays alerts on an LCD.

## Goal
Learn how to implement a sliding-window peak-to-peak detector to measure AC-coupled vibration signals without using blocking loops, write data to an SD card, and trigger warning alarms on threshold breaches.

## What You Will Build
A structural health monitor that tracks vibrations on three axes (or locations) using analog vibration sensors on ADC0, ADC1, and ADC2. The code tracks the maximum and minimum values in a continuous sliding time window. It calculates the vibration amplitude (Peak-to-Peak) for each axis. If the vibration on any sensor exceeds a set safety threshold, the system sounds an alarm buzzer, lights a warning LED, and logs a timestamped warning entry to a file on the SD card.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| 3x Analog Vibration Sensors | `vibration` | Yes | Yes |
| SD Card Reader Module | `sd_spiv3` | Yes | Yes |
| I2C LCD (16x2) | `lcd_i2c` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Warning LED | `led` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Vib Sensor 1 | OUT (analog) | ADC0 | Orange | Sensor 1 (GP26) |
| Vib Sensor 2 | OUT (analog) | ADC1 | Yellow | Sensor 2 (GP27) |
| Vib Sensor 3 | OUT (analog) | ADC2 | Blue | Sensor 3 (GP28) |
| SD Card Reader | CS | GPIO 8 | Purple | SPI Chip Select |
| SD Card Reader | SCK | GPIO 5 | Blue | SPI SCK |
| SD Card Reader | MOSI | GPIO 14 | Yellow | SPI COPI |
| SD Card Reader | MISO | GPIO 15 | Green | SPI CIPO |
| SD Card Reader | VCC | 5V | Red | Power |
| SD Card Reader | GND | GND | Black | Ground |
| I2C LCD (16x2) | SDA | GPIO 17 | Green | SDA0 |
| I2C LCD (16x2) | SCL | GPIO 16 | Yellow | SCL0 |
| Active Buzzer | + | GPIO 7 | Gray | Audio alarm pin |
| Active Buzzer | - | GND | Black | Ground |
| Red LED | Anode | GPIO 6 | Red | Alarm warning indicator |
| Red LED | Cathode | GND | Black | Ground |

> **Wiring tip:** Since the default buzzer and warning LED mappings (GPIO 14 and 15) conflict with the shared SPI pins used by the SD Card reader (MOSI and MISO), we move the Buzzer to GPIO 7 and the Warning LED to GPIO 6.

## Code
```cpp
// Vibration Structural Health Monitor - VEGA ARIES v3
#include <SPI.h>
#include <SD.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int SD_CS = 8;
const int SENSOR_0 = ADC0;
const int SENSOR_1 = ADC1;
const int SENSOR_2 = ADC2;
const int BUZZER_PIN = 7;
const int WARNING_LED = 6;

LiquidCrystal_I2C lcd(0x27, 16, 2);

// Peak-to-Peak Window Variables
int maxVal0 = 0, minVal0 = 1023;
int maxVal1 = 0, minVal1 = 1023;
int maxVal2 = 0, minVal2 = 1023;

int amp0 = 0;
int amp1 = 0;
int amp2 = 0;

unsigned long lastWindowTime = 0;
unsigned long lastLogTime = 0;
const int WINDOW_SIZE = 200; // Sample peak-to-peak in 200ms windows
const int ALERT_THRESHOLD = 180; // Vibration intensity alert limit

void setup() {
  Serial.begin(115200);
  SPI.begin();
  
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(WARNING_LED, OUTPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Structural H.M.");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  
  SD.begin(SD_CS);
  
  lastWindowTime = millis();
  lastLogTime = millis();
}

void loop() {
  // 1. High-frequency analog sampling to catch fast vibration peaks
  int raw0 = analogRead(SENSOR_0);
  int raw1 = analogRead(SENSOR_1);
  int raw2 = analogRead(SENSOR_2);
  
  // Track Min and Max in the current time-slice window
  if (raw0 > maxVal0) maxVal0 = raw0;
  if (raw0 < minVal0) minVal0 = raw0;
  
  if (raw1 > maxVal1) maxVal1 = raw1;
  if (raw1 < minVal1) minVal1 = raw1;
  
  if (raw2 > maxVal2) maxVal2 = raw2;
  if (raw2 < minVal2) minVal2 = raw2;
  
  unsigned long currentTime = millis();
  
  // 2. Evaluate Window Amplitude every 200ms
  if (currentTime - lastWindowTime >= WINDOW_SIZE) {
    lastWindowTime = currentTime;
    
    // Calculate Peak-to-Peak amplitudes
    amp0 = maxVal0 - minVal0;
    amp1 = maxVal1 - minVal1;
    amp2 = maxVal2 - minVal2;
    
    // Reset window min/max values
    maxVal0 = 0; minVal0 = 1023;
    maxVal1 = 0; minVal1 = 1023;
    maxVal2 = 0; minVal2 = 1023;
    
    // Check for exceeding the vibration safety limit
    bool alertActive = false;
    if (amp0 > ALERT_THRESHOLD || amp1 > ALERT_THRESHOLD || amp2 > ALERT_THRESHOLD) {
      alertActive = true;
      digitalWrite(BUZZER_PIN, HIGH);
      digitalWrite(WARNING_LED, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
      digitalWrite(WARNING_LED, LOW);
    }
    
    // Log immediately to SD card if a vibration breach occurs
    if (alertActive) {
      File logFile = SD.open("vib_alert.txt", FILE_WRITE);
      if (logFile) {
        logFile.print("ALERT! Timestamp: ");
        logFile.print(currentTime / 1000.0);
        logFile.print("s | Axis 0: ");
        logFile.print(amp0);
        logFile.print(" | Axis 1: ");
        logFile.print(amp1);
        logFile.print(" | Axis 2: ");
        logFile.println(amp2);
        logFile.close();
      }
      
      // Print alert immediately to serial
      Serial.print("STRUCTURAL WARNING: ");
      Serial.print("A0:"); Serial.print(amp0);
      Serial.print(" A1:"); Serial.print(amp1);
      Serial.print(" A2:"); Serial.println(amp2);
    }
  }
  
  // 3. Update LCD and log baseline trends to SD card every 2 seconds
  if (currentTime - lastLogTime >= 2000) {
    lastLogTime = currentTime;
    
    // Log baseline values to CSV
    File csvFile = SD.open("vib_trend.csv", FILE_WRITE);
    if (csvFile) {
      csvFile.print(currentTime / 1000.0);
      csvFile.print(",");
      csvFile.print(amp0);
      csvFile.print(",");
      csvFile.print(amp1);
      csvFile.print(",");
      csvFile.println(amp2);
      csvFile.close();
    }
    
    // Display updates
    lcd.clear();
    lcd.setCursor(0, 0);
    lcd.print("A0:");
    lcd.print(amp0);
    lcd.print(" A1:");
    lcd.print(amp1);
    
    lcd.setCursor(0, 1);
    lcd.print("A2:");
    lcd.print(amp2);
    if (amp0 > ALERT_THRESHOLD || amp1 > ALERT_THRESHOLD || amp2 > ALERT_THRESHOLD) {
      lcd.print(" *WARN*");
    } else {
      lcd.print(" Ok");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **3x Vibration Sensors**, **SD Card Reader**, **I2C LCD**, **Buzzer**, and **LED** onto the canvas.
2. Connect the sensors to **ADC0**, **ADC1**, and **ADC2**. Connect SD card and LCD matching the wiring table.
3. Paste the code into the editor and choose **Interpreted Mode**.
4. Click **Run**.
5. Manually adjust the vibration sliders on the sensor widgets to trigger alarms and check SD card files.

## Expected Output
Serial Monitor:
```
STRUCTURAL WARNING: A0:192 A1:14 A2:30
STRUCTURAL WARNING: A0:240 A1:120 A2:85
Baseline Logged: A0:12 A1:15 A2:8
```

## Expected Canvas Behavior
* On running, the LCD displays current peak-to-peak readings like "A0:12 A1:15" and "A2:8 Ok".
* If you drag any of the vibration sensor input sliders rapidly up and down (to simulate high peak-to-peak AC vibration), the buzzer beeps and the LED lights up.
* The LCD displays "*WARN*" beside the values.
* The SD card adds records to `vib_trend.csv` and warning alerts to `vib_alert.txt`.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `maxVal0` / `minVal0` | High-speed tracking variables updated in the loop to identify peak amplitudes. |
| `currentTime - lastWindowTime >= 200` | Limits amplitude evaluation cycles to 200ms windows to capture mechanical vibrations. |
| `amp0 = maxVal0 - minVal0` | Peak-to-Peak subtraction removes DC bias offsets from the piezo-ceramic sensors. |
| `amp0 > ALERT_THRESHOLD` | Threshold comparison triggers real-time warning indicators. |
| `SD.open("vib_alert.txt", ...)` | Records timestamps and magnitudes of critical safety limits breaches. |

## Hardware & Safety Concept
* **DC Bias Removal**: Piezoelectric vibration sensors are AC-coupled. They output a baseline voltage of VCC/2 (e.g. 1.65V) when idle. Subtracting the minimum from the maximum inside a time window removes this DC bias offset automatically.
* **Early Earthquake Warning**: In seismic monitoring, high-frequency P-waves arrive before destructive S-waves. Implementing fast peak detectors allows triggering shutdown systems before structural failures occur.

## Try This! (Challenges)
1. **Dynamic Alert Duration**: Once an alert triggers, keep the warning LED on for at least 5 seconds, even if the vibration immediately subsides.
2. **Frequency Estimator**: Count how many times the raw sensor input crosses the center line (512) inside the 200ms window to estimate the vibration frequency (Hz).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Vibration amplitude reads close to 1023 constantly | Sensor terminal floating or disconnected | Check that the sensor has common ground and power. |
| Alarm does not trigger during shakes | Window size too large or threshold too high | Reduce the threshold (ALERT_THRESHOLD) or decrease the window size in the code. |
| SD Card fails on high vibration | Mechanical connector vibrations | Ensure the SD module slot contacts are clean. Secure the SD card using tape or vibration mounts in physical builds. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [137 - SD Card Temperature Logger](../advanced/137-sd-card-temperature-logger.md)
- [142 - Tilt Vault Alarm](../advanced/142-tilt-vault-alarm.md)
- [180 - Automated Greenhouse](180-automated-greenhouse.md)
