# 83 - DS18B20 Multi-Sensor LCD HUD

Interrogate multiple DS18B20 temperature probes connected to a single OneWire bus pin and display their readings simultaneously on a 16x2 I2C LCD.

## Goal
Learn how the Dallas OneWire bus addresses multiple devices individually on the same GPIO pin, and display multi-sensor telemetry on an LCD screen without using loops or array buffers.

## What You Will Build
Two DS18B20 temperature sensors are connected to GPIO 12, and a 16x2 I2C LCD is connected to the hardware I2C0 interface of the ARIES v3 board. The LCD displays the temperature of Sensor 1 on Row 1 and Sensor 2 on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS18B20 Temperature Sensor (2) | `ds18b20` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |
| 4.7 kΩ Resistor (pull-up) | `resistor` | No | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS18B20 Sensor 1 | VCC (Red) | 3V3 | Red | Power (shared) |
| DS18B20 Sensor 1 | GND (Black) | GND | Black | Ground (shared) |
| DS18B20 Sensor 1 | DATA (Yellow) | GPIO 12 | Yellow | OneWire data bus (shared) |
| DS18B20 Sensor 2 | VCC (Red) | 3V3 | Red | Power (shared) |
| DS18B20 Sensor 2 | GND (Black) | GND | Black | Ground (shared) |
| DS18B20 Sensor 2 | DATA (Yellow) | GPIO 12 | Yellow | OneWire data bus (shared) |
| I2C LCD | VCC | 5V | Red | LCD power supply |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| Pull-up Resistor | VCC to DATA | — | — | Place 4.7k resistor between VCC and DATA pins |

> **Wiring tip:** Both DS18B20 sensors connect in parallel to the exact same DATA pin (GPIO 12). Only one 4.7 kΩ pull-up resistor is needed for the entire bus line.

## Code
```cpp
// DS18B20 Multi-Sensor LCD HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONEWIRE_BUS = 12;

OneWire oneWire(ONEWIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("Sensors Init...");
  
  sensors.begin();
  delay(1000);
  lcd.clear();
}

void loop() {
  sensors.requestTemperatures();
  
  // Read temperature from the first sensor (index 0)
  float temp1 = sensors.getTempCByIndex(0);
  // Read temperature from the second sensor (index 1)
  float temp2 = sensors.getTempCByIndex(1);
  
  // Display Sensor 1 on Line 1
  lcd.setCursor(0, 0);
  if (temp1 == DEVICE_DISCONNECTED_C) {
    lcd.print("S1: Disconnected");
  } else {
    lcd.print("S1 Temp: ");
    lcd.print(temp1, 1);
    lcd.print(" C     "); // Trailing spaces to overwrite old text
  }
  
  // Display Sensor 2 on Line 2
  lcd.setCursor(0, 1);
  if (temp2 == DEVICE_DISCONNECTED_C) {
    lcd.print("S2: Disconnected");
  } else {
    lcd.print("S2 Temp: ");
    lcd.print(temp2, 1);
    lcd.print(" C     ");
  }
  
  delay(1500); // 1.5-second refresh interval
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board, **two DS18B20 Sensors**, and an **I2C LCD Display** onto the canvas.
2. Connect both DS18B20 DATA pins to **GPIO 12**.
3. Connect the LCD SDA/SCL pins to **SDA0 (GP17)** and **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.
7. Change the sliders on both DS18B20 widgets and watch the LCD display update.

## Expected Output
LCD Screen:
```
S1 Temp: 24.5 C
S2 Temp: 28.1 C
```

## Expected Canvas Behavior
* Row 1 shows Sensor 1's temperature; Row 2 shows Sensor 2's temperature.
* Adjusting the sliders on the respective DS18B20 widgets updates the LCD display values independently on the next loop cycle.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `sensors.getTempCByIndex(0)` | Reads the temperature value of the first physical sensor detected on the OneWire bus. |
| `sensors.getTempCByIndex(1)` | Reads the temperature value of the second physical sensor detected on the OneWire bus. |
| `temp1 == DEVICE_DISCONNECTED_C` | Checks if Sensor 1 is disconnected or has failed. |
| `lcd.setCursor(0, 1)` | Switches cursor to the second row for Sensor 2's output. |

## Hardware & Safety Concept
* **Bus Device Enumeration**: When `sensors.begin()` runs, the library scans the OneWire bus. It detects each unique 64-bit ROM address and stores them in memory. The index parameter (0, 1) refers to these addresses sorted numerically. This allows up to dozens of sensors to connect to a single microcontroller pin, which is highly useful in multi-zone temperature logging (e.g. soil depth layers or greenhouse zones).

## Try This! (Challenges)
1. **Delta Temp Display**: Modify the display format so Row 1 shows Sensor 1's temperature and Row 2 shows the temperature difference between the two sensors (`S1 - S2`).
2. **Dynamic Alarm warning**: Turn on the onboard Red LED (`LED_R`) if the temperature difference between the two sensors exceeds 5.0 °C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD prints "S2: Disconnected" | Second sensor not connected or bus error | Verify both sensors share the same DATA line (GPIO 12). |
| Both show disconnected | Missing pull-up resistor | Connect a 4.7 kΩ pull-up resistor between GPIO 12 and 3.3V. |
| Indexes are swapped | Address sorting changes | The library sorts addresses numerically. If a sensor is replaced, index order may swap. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - DS18B20 One-Wire Temp Sensor Serial Logs](81-ds18b20-one-wire-temp-serial.md)
- [82 - DS18B20 Temperature Alarm](82-ds18b20-temperature-alarm.md)
- [73 - DHT22 Temperature LCD HUD](73-dht22-temperature-lcd-hud.md)
