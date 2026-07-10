# 83 - Pico DS18B20 LCD

Read temperature values from two DS18B20 sensors on the same One-Wire bus and display them on an LCD.

## Goal
Learn how to address and read multiple sensors sharing a single digital input pin and print their values to a 16x2 LCD.

## What You Will Build
A dual-point thermometer HUD:
- **Sensor 1 & Sensor 2 (GP12 One-Wire bus)**: Share the same data wire.
- **16x2 I2C LCD (GP4, GP5)**: Displays Sensor 1 temperature on row 0 (e.g. "T1: 24.5 C") and Sensor 2 on row 1 (e.g. "T2: 26.1 C").

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temp Sensors | `ds18b20` | Yes (two sensors) | Yes (two waterproof probes) |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 #1 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| DS18B20 #2 | DQ (Data) | GP12 | Shared One-Wire bus data line |
| Both Sensors | VCC | 3.3V | Parallel power (with 4.7k pull-up to 3.3V) |
| Both Sensors | GND | GND | Shared ground return |
| I2C LCD | VCC | 5V | Display power |
| I2C LCD | SDA | GP4 | I2C Data |
| I2C LCD | SCL | GP5 | I2C Clock |
| I2C LCD | GND | GND | Ground return |

## Code
```cpp
#include <Wire.h>
#include <OneWire.h>
#include <DallasTemperature.h>
#include <LiquidCrystal_I2C.h>

const int ONE_WIRE_BUS = 12;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.setSDA(4);
  Wire.setSCL(5);
  Wire.begin();

  sensors.begin();
  lcd.init();
  lcd.backlight();
  
  lcd.setCursor(0, 0);
  lcd.print("Dual Temp Probe");
  delay(1000);
}

void loop() {
  sensors.requestTemperatures();
  
  // Read both sensors by index
  float temp1 = sensors.getTempCByIndex(0);
  float temp2 = sensors.getTempCByIndex(1);

  lcd.clear();

  // Print Sensor 1 on Row 0
  lcd.setCursor(0, 0);
  if (temp1 != DEVICE_DISCONNECTED_C) {
    lcd.print("Probe 1: ");
    lcd.print(temp1);
    lcd.print(" C");
  } else {
    lcd.print("Probe 1: Error");
  }

  // Print Sensor 2 on Row 1
  lcd.setCursor(0, 1);
  if (temp2 != DEVICE_DISCONNECTED_C) {
    lcd.print("Probe 2: ");
    lcd.print(temp2);
    lcd.print(" C");
  } else {
    lcd.print("Probe 2: Error");
  }

  delay(1500); // Update once per 1.5 seconds
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **two DS18B20 Sensors**, and **I2C LCD** onto the canvas.
2. Connect both DS18B20 sensors' data pins to **GP12** in parallel (add a 4.7k resistor to 3V3).
3. Connect LCD: **VCC** to **5V**, **SDA** to **GP4**, **SCL** to **GP5**, **GND** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Adjust the temp sliders on both sensors and watch the LCD update.

## Expected Output

Terminal:
```
Simulation active. LCD rendering dual DS18B20 temperatures.
```

## Expected Canvas Behavior
* Row 0: `Probe 1: 24.50 C`
* Row 1: `Probe 2: 26.10 C`

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensors.getTempCByIndex(1)` | Identifies the second sensor on the shared bus by its unique address index and requests its temperature. |

## Hardware & Safety Concept: Address Enumeration
When the One-Wire library initializes, it scans the bus wire and enumerates all connected devices. It sorts them based on their unique 64-bit internal ROM addresses, assigning them indices starting from 0. On real hardware, to make sure you know which physical probe is "Sensor 0" and which is "Sensor 1", you must plug them in one by one and record their unique address codes.

## Try This! (Challenges)
1. **Delta Display**: Add code to calculate the difference (delta) between the two sensors and display it on the LCD screen.
2. **Alert indicator**: Flash a warning LED on GP15 if the difference between the two probes is greater than 5.0°C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Row 1 shows "Probe 2: Error" | Second sensor disconnected | Confirm that the data pins of both sensors are tied to the exact same physical node on GP12. |

## Mode Notes
This multi-device I2C and One-Wire project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [58 - Pico LCD Print](58-pico-lcd-print.md)
- [81 - Pico DS18B20 Serial](81-pico-ds18b20-serial.md)
