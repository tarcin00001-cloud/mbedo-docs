# 83 - ESP32 DS18B20 Multi-Sensor LCD HUD

Connect multiple DS18B20 temperature sensors in parallel to a single GPIO pin and display their independent readings on a 16x2 I2C LCD screen.

## Goal
Learn how the OneWire bus addresses multiple devices on the same wire, retrieve temperatures by index, and display dual-sensor data on a character LCD.

## What You Will Build
Two DS18B20 sensors are connected in parallel to GPIO 4. The ESP32 reads both sensors every 2 seconds. The 16x2 I2C LCD displays the temperature of Sensor 1 on Row 1 and the temperature of Sensor 2 on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS18B20 Sensors (2) | `ds18b20` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |
| 4.7 kΩ Resistor | `resistor` | No | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Both Sensors | VCC (Red) | 3V3 | Red | Parallel power connection |
| Both Sensors | GND (Black) | GND | Black | Parallel ground reference |
| Both Sensors | DATA (Yellow) | GPIO4 | Yellow | Parallel OneWire data connection |
| Pull-up Resistor | VCC to DATA | — | — | One single resistor pulls up the entire bus |
| I2C LCD | VCC | 5V (Vin) | Red | LCD power |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock |

> **Wiring tip:** When connecting multiple OneWire devices, wire them in parallel (daisy-chain style): connect all VCC wires together, all GND wires together, and all DATA wires together. You only need **one single 4.7 kΩ pull-up resistor** on the shared DATA wire, near the microcontroller.

## Code
```cpp
// DS18B20 Multi-Sensor LCD HUD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONEWIRE_BUS = 4;

OneWire oneWire(ONEWIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

int deviceCount = 0;

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("OneWire Scanner");
  
  sensors.begin();
  deviceCount = sensors.getDeviceCount();
  
  lcd.setCursor(0, 1);
  lcd.print("Found: ");
  lcd.print(deviceCount);
  lcd.print(" sensors");
  
  delay(2000);
  lcd.clear();
}

void loop() {
  sensors.requestTemperatures();
  
  // Read Sensor 1 (Index 0)
  float temp1 = sensors.getTempCByIndex(0);
  // Read Sensor 2 (Index 1)
  float temp2 = (deviceCount >= 2) ? sensors.getTempCByIndex(1) : DEVICE_DISCONNECTED_C;
  
  // Display Sensor 1 Temp on Row 1
  lcd.setCursor(0, 0);
  if (temp1 == DEVICE_DISCONNECTED_C) {
    lcd.print("S1: Error      ");
  } else {
    lcd.print("S1 Temp: ");
    lcd.print(temp1, 1);
    lcd.print((char)223);
    lcd.print("C   ");
  }
  
  // Display Sensor 2 Temp on Row 2
  lcd.setCursor(0, 1);
  if (deviceCount < 2) {
    lcd.print("S2: Not Found  ");
  } else if (temp2 == DEVICE_DISCONNECTED_C) {
    lcd.print("S2: Error      ");
  } else {
    lcd.print("S2 Temp: ");
    lcd.print(temp2, 1);
    lcd.print((char)223);
    lcd.print("C   ");
  }
  
  Serial.print("S1: "); Serial.print(temp1, 1);
  Serial.print(" °C | S2: "); Serial.print(temp2, 1); Serial.println(" °C");
  
  delay(2000); // 2-second update interval
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, two **DS18B20 Sensors**, and one **16x2 I2C LCD** onto the canvas.
2. Connect both sensor DATA pins to **GPIO4**. Wire the LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust the temperature sliders on both DS18B20 sensor widgets on the canvas, and watch both lines of the LCD update independently.

## Expected Output
Serial Monitor:
```
S1: 24.5 °C | S2: 30.2 °C
S1: 22.0 °C | S2: 31.8 °C
```

LCD Display:
```
S1 Temp: 24.5°C
S2 Temp: 30.2°C
```

## Expected Canvas Behavior
* At startup, the LCD scanner prints the count of detected sensors.
* Sensor 1's temperature slider adjustments change the top LCD line.
* Sensor 2's temperature slider adjustments change the bottom LCD line.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `sensors.getDeviceCount()` | Scans the OneWire bus at startup to determine how many devices are responding. |
| `sensors.getTempCByIndex(0)` | Requests temperature from index 0. Index mapping is sorted automatically by the unique addresses. |
| `(deviceCount >= 2) ? ... : ...` | Safeguards the code: only requests index 1 if at least two devices were detected. |

## Hardware & Safety Concept: Address Sorting and Bus Routing
The `DallasTemperature` library lists sensors on the bus sorted by their unique 64-bit ROM addresses. This sorting is deterministic but not related to physical placement. In production, developers scan and register the hardcoded serial addresses of each sensor to map them to physical locations (e.g. "Sensor A = Inlet", "Sensor B = Outlet").

## Try This! (Challenges)
1. **Address Scanner**: Print the hex address of all connected sensors on the Serial Monitor during startup.
2. **Delta Temperature HUD**: Display the temperature difference (\(\Delta T\)) between Sensor 1 and Sensor 2 on the bottom row.
3. **Rotating HUD**: If three or more sensors are connected, scroll through the sensors automatically every 3 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD displays "S2: Not Found" | Second sensor wire disconnected or parallel connection missing | Verify both sensor DATA pins are connected to GPIO 4 |
| Only one sensor detected at boot | Bad physical connection or lack of bus pull-up | Verify the 4.7 kΩ pull-up is present and connecting VCC to DATA |
| S1 and S2 displays are swapped | Address indexing order | Flip the sensor index numbers (0 and 1) in your display mapping |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [81 - ESP32 DS18B20 One-Wire Temp Sensor Serial Logs](81-esp32-ds18b20-onewire-temp-sensor-serial-logs.md)
- [82 - ESP32 DS18B20 Temperature Alarm](82-esp32-ds18b20-temperature-alarm.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
