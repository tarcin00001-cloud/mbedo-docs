# 163 - Water Quality Station

Monitor water quality by reading a pH sensor on ADC0/GP26 and a DS18B20 waterproof temperature sensor on GPIO 12, then displaying both values on an I2C LCD and printing structured logs to the Serial Monitor.

## Goal
Learn to integrate an analog pH electrode module, a 1-Wire digital temperature sensor, and an I2C character LCD into a multi-sensor environmental monitoring station. Understand pH sensor calibration offsets and OneWire communication.

## What You Will Build
A pH electrode module (which outputs a 0–3.3 V analog signal proportional to pH 0–14) connects to ADC0 (GP26). A DS18B20 waterproof temperature probe on GPIO 12 provides water temperature data using the OneWire and DallasTemperature libraries. An I2C LCD (16×2, address 0x27) displays current pH and temperature readings. The Serial Monitor prints a formatted log every 2 seconds.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog pH Sensor Module | `ph_sensor` | Yes | Yes |
| DS18B20 Waterproof Temperature Sensor | `ds18b20` | Yes | Yes |
| 4.7 kΩ Pull-up Resistor | `resistor` | No | Yes |
| I2C LCD 16×2 (PCF8574 backpack) | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| pH Sensor Module | VCC | 5V | Red | Module power supply |
| pH Sensor Module | GND | GND | Black | Common ground |
| pH Sensor Module | Po (analog out) | ADC0 / GP26 | Yellow | pH voltage output |
| DS18B20 | VCC (red wire) | 3V3 | Red | Sensor power |
| DS18B20 | DATA (yellow wire) | GPIO 12 | Yellow | OneWire data line |
| DS18B20 | GND (black wire) | GND | Black | Common ground |
| Pull-up Resistor | VCC to DATA | — | — | 4.7 kΩ between 3V3 and GPIO 12 |
| I2C LCD | VCC | 5V | Red | LCD backlight and logic power |
| I2C LCD | GND | GND | Black | Common ground |
| I2C LCD | SDA | SDA (GPIO 4) | Blue | I2C data line |
| I2C LCD | SCL | SCL (GPIO 5) | Green | I2C clock line |

> **Wiring tip:** The pH sensor module must be calibrated to your specific electrode. Most modules include a trimmer potentiometer that sets the output to 2.5 V at pH 7.0 (neutral). Before use, immerse the probe in pH 4.0 buffer solution, read the ADC value, then repeat with pH 7.0 and pH 10.0 solutions to build a linear calibration. The `PH_OFFSET` constant in the code accounts for deviations from the ideal 0 V = pH 0 to 3.3 V = pH 14 mapping.

## Code
```cpp
// Water Quality Station
// pH: ADC0/GP26 | DS18B20: GPIO 12 | I2C LCD: SDA/SCL

#include <OneWire.h>
#include <DallasTemperature.h>
#include <LiquidCrystal_I2C.h>

#define PH_PIN    26   // ADC0/GP26
#define OW_PIN    12   // DS18B20 OneWire bus

// pH calibration constants
// Adjust PH_OFFSET until pH 7.0 buffer reads 7.00
const float PH_OFFSET  = 0.0f;
const float VREF       = 3.3f;
const float ADC_MAX    = 4095.0f;

OneWire oneWire(OW_PIN);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

float phValue   = 0.0f;
float tempC     = 0.0f;
float phVoltage = 0.0f;

void setup() {
  Serial.begin(115200);

  sensors.begin();
  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("Water Quality   ");
  lcd.setCursor(0, 1);
  lcd.print("Station Ready   ");

  Serial.println("=== Water Quality Station ===");
  Serial.println("pH | Temp(C) | Status");
  delay(2000);
}

void loop() {
  // Read pH analog voltage and convert to pH scale (0-14)
  int rawPH   = analogRead(PH_PIN);
  phVoltage   = (rawPH / ADC_MAX) * VREF;
  // Linear mapping: 0 V = pH 0, 3.3 V = pH 14, plus calibration offset
  phValue     = (phVoltage / VREF) * 14.0f + PH_OFFSET;

  // Clamp pH to valid range
  if (phValue < 0.0f)  phValue = 0.0f;
  if (phValue > 14.0f) phValue = 14.0f;

  // Read DS18B20 temperature
  sensors.requestTemperatures();
  tempC = sensors.getTempCByIndex(0);

  // Determine water quality status from pH
  const char* status = "Neutral ";
  if (phValue < 6.5f)       status = "Acidic  ";
  else if (phValue > 8.5f)  status = "Alkaline";

  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("pH: ");
  lcd.print(phValue, 2);
  lcd.print("  ");
  lcd.print(status);

  lcd.setCursor(0, 1);
  lcd.print("Temp: ");
  lcd.print(tempC, 1);
  lcd.print(" C       ");

  // Serial log
  Serial.print("pH=");
  Serial.print(phValue, 2);
  Serial.print(" | T=");
  Serial.print(tempC, 1);
  Serial.print(" C | ");
  Serial.println(status);

  delay(2000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board onto the canvas.
2. Add a **Potentiometer** component and connect its wiper output to **ADC0/GP26** (simulating the pH sensor analog voltage).
3. Add a **DS18B20** component and connect its DATA pin to **GPIO 12**.
4. Add an **I2C LCD** component; connect **SDA** to the ARIES SDA pin and **SCL** to the SCL pin.
5. Paste the code into the editor.
6. Select **Interpreted Mode** from the simulation dropdown.
7. Click **Run**.
8. Adjust the potentiometer slider to simulate different pH voltages and the DS18B20 temperature slider to change the water temperature; observe both LCD and Serial Monitor updates.

## Expected Output
Serial Monitor:
```
=== Water Quality Station ===
pH | Temp(C) | Status
pH=7.02 | T=24.5 C | Neutral
pH=5.10 | T=24.5 C | Acidic
pH=9.15 | T=25.0 C | Alkaline
```

LCD display:
```
pH: 7.02  Neutral
Temp: 24.5 C
```

## Expected Canvas Behavior
* The I2C LCD widget updates every 2 seconds with current pH and temperature values.
* The status label (`Neutral`, `Acidic`, or `Alkaline`) changes on the LCD and in the Serial Monitor as the potentiometer crosses the threshold values.
* The DS18B20 temperature slider change is reflected in the next Serial Monitor line.
* When the potentiometer is centred (≈ 2.5 V = pH 7), the status shows "Neutral".

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `#include <OneWire.h>` | Imports the OneWire protocol library for communicating with the DS18B20. |
| `#include <DallasTemperature.h>` | Imports the DallasTemperature library that wraps OneWire for DS18B20 sensors. |
| `#include <LiquidCrystal_I2C.h>` | Imports the I2C LCD library for the PCF8574-backed 16×2 display. |
| `phVoltage = (rawPH / ADC_MAX) * VREF` | Converts 12-bit ADC count to voltage (0–3.3 V). |
| `phValue = (phVoltage / VREF) * 14.0f + PH_OFFSET` | Linear mapping from voltage range to pH 0–14 scale with calibration offset. |
| `sensors.requestTemperatures()` | Sends the 1-Wire command to trigger all DS18B20 sensors to convert temperature. |
| `sensors.getTempCByIndex(0)` | Reads the temperature from the first (index 0) sensor on the OneWire bus. |
| `lcd.setCursor(0, 0); lcd.print(...)` | Positions the cursor and writes text to the first row of the LCD. |

## Hardware & Safety Concept
* **pH Electrode Chemistry**: A pH electrode is a galvanic cell that develops a Nernst potential proportional to the logarithm of hydrogen ion activity (pH). The potential changes approximately 59.16 mV per pH unit at 25 °C (Nernst slope). The pH module amplifies this millivolt-level signal and offsets it to produce a measurable 0–3.3 V output. Temperature affects the Nernst slope, which is why the DS18B20 reading can be used for a temperature-compensated pH calculation.
* **DS18B20 OneWire**: The DS18B20 communicates over a single-wire bidirectional protocol. Up to 127 sensors can share one data line using unique 64-bit ROM addresses. The 4.7 kΩ pull-up resistor holds the bus HIGH in idle state, which is required for the protocol to function.
* **Probe Storage**: Always store pH electrodes submerged in pH 4.0 storage solution or saturated KCl. Never store in distilled or DI water as this depletes the electrolyte from the reference junction.

## Try This! (Challenges)
1. **Temperature-Compensated pH**: Multiply the Nernst slope correction by temperature: `slope = 0.03354 + (0.00021 × tempC)` and recalculate pH using `phValue = 7.0 + (phVoltage - 1.65f) / (slope * -14.0f / VREF)` for a more accurate reading in variable-temperature water bodies.
2. **Alarm Threshold Alert**: Toggle GPIO 15 (warning LED) HIGH when pH falls below 6.0 or above 9.0, providing a visual out-of-range alarm for a fish tank or hydroponics reservoir.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| pH reads 0.00 always | ADC0/GP26 not connected or potentiometer slider at minimum | Move slider upward; verify wiring from module Po pin to ADC0/GP26. |
| Temperature reads -127°C | DS18B20 not found on OneWire bus, or missing pull-up | Check that the 4.7 kΩ pull-up is present on GPIO 12; re-seat sensor connections. |
| LCD shows blank rows | Wrong I2C address or SDA/SCL swapped | Try I2C address 0x3F if 0x27 fails; verify SDA and SCL pin assignment. |
| pH reads non-linearly | Electrode needs calibration | Use pH buffer solutions to determine the actual voltage-to-pH mapping and update `PH_OFFSET`. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [164 - Water Flow Rate Station](164-water-flow-rate-station.md)
- [161 - Solar Tracker Dual Axis Controller](161-solar-tracker-dual-axis.md)
- [166 - Smart Fan with Temperature Hysteresis](166-smart-fan.md)
