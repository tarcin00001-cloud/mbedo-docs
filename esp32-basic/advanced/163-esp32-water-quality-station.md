# 163 - ESP32 Water Quality Station

Build a water quality monitoring station that measures pH levels using an analog pH sensor, captures precise water temperature using a DS18B20 OneWire sensor, applies temperature compensation corrections to the pH readings, and displays data on an LCD.

## Goal
Learn how to read OneWire digital temperature sensors, calibrate and compensate analog pH probe curves, and implement safety threshold alarms.

## What You Will Build
An analog pH sensor is connected to GPIO 34. A waterproof DS18B20 temperature sensor is on GPIO 4. A 16x2 I2C LCD displays pH and temperature. An active buzzer is on GPIO 15. If the pH level drifts outside the safe drinking water window (6.5 to 8.5 pH), the buzzer sounds an alarm.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Analog pH Sensor Probe | `ph_sensor` | Yes | Yes |
| DS18B20 Temperature Sensor | `ds18b20` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| 4.7 kΩ Resistor (for OneWire) | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| pH Sensor Board | OUT (Analog) | GPIO34 | Yellow | Analog pH input |
| pH Sensor Board | VCC / GND | 5V / GND | Red / Black | Power rails |
| DS18B20 Sensor | DQ (Data) | GPIO4 | Green | OneWire data |
| 4.7 kΩ Resistor | Leg 1 / Leg 2 | GPIO4 / 3V3 | White | Pull-up for OneWire |
| DS18B20 Sensor | VCC / GND | 3V3 / GND | Red / Black | Sensor power |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| Active Buzzer | VCC (+) | GPIO15 | Blue | Warning buzzer |

> **Wiring tip:** The DS18B20 requires a 4.7 kΩ pull-up resistor between the DQ (data) pin (GPIO 4) and the 3.3V rail. The pH sensor board is powered by 5V Vin for stable reference.

## Code
```cpp
// Water Quality Station (pH + DS18B20 + LCD)
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int PH_PIN = 34;
const int ONE_WIRE_BUS = 4;
const int BUZZER_PIN = 15;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

// pH Sensor Calibration parameters
// Standard sensor outputs 2.5V at pH 7.0 (when powered from 5V, divided to 1.65V for ESP32)
const float PH_7_VOLTAGE = 1.65; 
const float PH_SLOPE = 0.18; // Voltage change per pH unit

// Safe water pH range limits
const float PH_SAFE_MIN = 6.5;
const float PH_SAFE_MAX = 8.5;

unsigned long lastSampleTime = 0;
const unsigned long SAMPLE_INTERVAL_MS = 2000; // Sample once every 2 seconds

float readTemperature() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);
  if (tempC == DEVICE_DISCONNECTED_C) return 25.0; // Default to 25C if sensor offline
  return tempC;
}

float readPH(float tempC) {
  int raw = analogRead(PH_PIN);
  float voltage = raw * (3.3 / 4095.0);
  
  // Temperature compensation factor:
  // Standard pH glass electrodes change slope by ~0.03 pH units per °C deviation from 25 °C
  float tempCompensation = (tempC - 25.0) * 0.03;
  
  // Calculate pH: 7.0 + (V_7 - V_read) / Slope + correction
  float phValue = 7.0 + ((PH_7_VOLTAGE - voltage) / PH_SLOPE) + tempCompensation;
  
  // Clamp to realistic bounds
  phValue = constrain(phValue, 0.0, 14.0);
  return phValue;
}

void setup() {
  Serial.begin(115200);
  
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  sensors.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Water Quality");
  lcd.setCursor(0, 1);
  lcd.print("Station Active");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  unsigned long now = millis();
  
  if (now - lastSampleTime >= SAMPLE_INTERVAL_MS) {
    float waterTemp = readTemperature();
    float waterPH = readPH(waterTemp);
    
    // Evaluate if pH is outside safe boundaries
    bool isUnsafe = (waterPH < PH_SAFE_MIN || waterPH > PH_SAFE_MAX);
    
    if (isUnsafe) {
      Serial.println("!! WARNING: UNSAFE WATER PH LEVEL !!");
      // sound warning buzzer
      digitalWrite(BUZZER_PIN, HIGH);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
    
    // Update LCD Screen
    lcd.setCursor(0, 0);
    lcd.print("pH Value: ");
    lcd.print(waterPH, 2);
    lcd.print("   "); // Clear trailing characters
    
    lcd.setCursor(0, 1);
    lcd.print("Temp: ");
    lcd.print(waterTemp, 1);
    lcd.print((char)223);
    lcd.print("C ");
    if (isUnsafe) lcd.print("ALERT");
    else lcd.print("OK   ");
    
    Serial.print("Temp: "); Serial.print(waterTemp, 1);
    Serial.print(" C | pH: "); Serial.print(waterPH, 2);
    Serial.println(isUnsafe ? " (UNSAFE)" : " (SAFE)");
    
    lastSampleTime = now;
  }
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **pH Sensor**, **DS18B20**, **Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Wire DS18B20 to **GPIO4**, pH to **GPIO34**, Buzzer to **GPIO15**, and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust pH slider. Set pH to 7.0. Watch the LCD print safe.
5. Slide the pH to 5.0 (acidic) or 9.0 (alkaline). Watch the buzzer turn ON and LCD display ALERT.

## Expected Output
Serial Monitor:
```
Temp: 24.5 C | pH: 7.12 (SAFE)
Temp: 24.5 C | pH: 5.80 (UNSAFE)
!! WARNING: UNSAFE WATER PH LEVEL !!
```

LCD Display (unsafe):
```
pH Value: 5.80
Temp: 24.5°C ALERT
```

## Expected Canvas Behavior
* The LCD displays the live pH and temperature.
* Adjusting the pH slider on the simulated widget changes the pH value on the LCD.
* If the pH slider goes below 6.5 or above 8.5, the buzzer widget turns ON (green) and the LCD displays "ALERT".

## Code Walkthrough
| Line | Math / Step |
| --- | --- |
| `sensors.getTempCByIndex(0)` | Reads the digital temperature from the first DS18B20 sensor found on the bus. |
| `(tempC - 25.0) * 0.03` | Calculates temperature compensation to adjust for the pH probe's sensitivity change. |
| `waterPH < PH_SAFE_MIN` | Monitors pH limits to sound the warning alarm if water is unsafe. |

## Hardware & Safety Concept: pH Probe Chemistry and Temperature Compensation
pH probes measure the hydrogen ion concentration ($H^+$) in a solution. The probe contains a reference electrode and a glass sensing membrane that generates a microvolt potential proportional to pH. The sensitivity (slope) of this potential depends on temperature (defined by the Nernst Equation). At 25 °C, the slope is 59.16 mV per pH unit. At other temperatures, the slope changes, requiring **temperature compensation** using a temperature sensor (like the DS18B20) to maintain accuracy.

## Try This! (Challenges)
1. **Interactive Calibration mode**: Add two buttons on GPIO 12 and 13. Holding Button A calibrates pH 7.0, and Button B calibrates pH 4.0.
2. **SD Card Data Logging**: Integrate an SD card reader (Project 137) to log pH and temperature to a CSV file.
3. **Acid/Base dosing relay**: Add two relays on GPIO 14 and 27 to control acid/base dosing pumps to keep the pH balanced.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Temperature reads -127 °C | OneWire bus connection lost or missing pull-up | Verify DS18B20 DQ pin is connected; ensure the 4.7 kΩ pull-up resistor is installed |
| pH readings are static or drift constantly | Calibration offset incorrect | Verify the pH sensor board is receiving a stable 5V supply |
| pH values change when current loads turn on | Ground loop noise | Ensure analog grounds are isolated from high-current motor grounds |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - ESP32 DS18B20 OneWire Temp Sensor Serial Logs](../intermediate/81-esp32-ds18b20-onewire-temp-sensor-serial-logs.md)
- [105 - ESP32 Thermostat Controller](105-esp32-thermostat-controller.md)
- [137 - ESP32 SD Card Temperature Logger](137-esp32-sd-card-temperature-logger.md)
