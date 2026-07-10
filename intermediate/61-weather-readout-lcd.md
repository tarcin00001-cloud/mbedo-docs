# 61 - Weather Readout LCD

Display temperature and barometric pressure from a BMP180 sensor on a 16x2 I2C LCD screen.

## Goal
Learn how to interface multiple I2C devices (BMP180 sensor and LCD display) on the same two-wire bus and display environmental values.

## What You Will Build
The Arduino reads temperature and pressure from the BMP180, formats them, and writes:
- Line 1: "Temp: 24.50 C"
- Line 2: "Pres: 1013.25 hPa"
The screen updates every 1 second.

**Why A4 and A5?** Since both the BMP180 sensor and the LCD display use I2C communication, they share pins A4 (SDA) and A5 (SCL) in parallel.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| LCD I2C 16x2 | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | Share SDA line |
| BMP180 Sensor | SCL | A5 | Share SCL line |
| BMP180 Sensor | GND | GND | Ground reference |
| LCD Module | VCC | 5V | Power supply |
| LCD Module | SDA | A4 | Share SDA line |
| LCD Module | SCL | A5 | Share SCL line |
| LCD Module | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>
#include <LiquidCrystal_I2C.h>

Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  
  // Initialize LCD
  lcd.init();
  lcd.backlight();
  lcd.clear();
  
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  lcd.setCursor(0, 1);
  lcd.print("Starting...");
  
  // Initialize BMP180
  if (!bmp.begin()) {
    lcd.clear();
    lcd.print("Sensor Error!");
    while (1) {}
  }
  
  delay(1500);
}

void loop() {
  float tempC = bmp.readTemperature();
  int32_t pressurePa = bmp.readPressure();
  float pressureHPa = pressurePa / 100.0;
  
  lcd.clear();
  
  // Line 1: Temperature
  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(tempC);
  lcd.print(" C");
  
  // Line 2: Pressure
  lcd.setCursor(0, 1);
  lcd.print("Pres: ");
  lcd.print(pressureHPa);
  lcd.print(" hPa");
  
  delay(1000); // Update every second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **BMP180 Sensor**, and **LCD I2C 16x2** onto the canvas.
2. Connect BMP180 **VCC** to Arduino **3.3V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
3. Connect LCD **VCC** to Arduino **5V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the BMP180, adjust the temperature and pressure sliders, and watch the values display on the LCD.

## Expected Output
The LCD screen component on the canvas lights up and displays:
```
Temp: 23.50 C
Pres: 1013.25 hPa
```
Values update dynamically as you move the sliders on the BMP180 sensor.

## Expected Canvas Behavior

| Temperature Input | Pressure Input | LCD Line 1 Display | LCD Line 2 Display |
| --- | --- | --- | --- |
| 28.5 C | 1008 hPa | "Temp: 28.50 C" | "Pres: 1008.00 hPa" |
| 15.0 C | 995 hPa | "Temp: 15.00 C" | "Pres: 995.00 hPa" |

The display updates automatically every 1 second.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `#include <Wire.h>` | Coordinates I2C communication transactions. |
| `lcd.print(pressureHPa)` | Formats and prints the pressure decimal value on the screen. |

## Hardware & Safety Concept: Shared I2C Buses
Since the I2C bus utilizes **address-based communication**, multiple devices can share the exact same two pins (SDA and SCL) in parallel without interference. 
- The BMP180 listens on address `0x77`.
- The LCD backpack listens on address `0x27`.
When the Arduino sends data to `0x27`, only the LCD processes it; the BMP180 ignores the transmission. This allows you to wire dozens of I2C devices together on a single bus.

## Try This! (Challenges)
1. **Fahrenheit Conversion**: Modify the code to display Fahrenheit on line 1.
2. **Storm Warning Display**: If the pressure drops below `1005` hPa, flash the message "STORM WARNING!" on the display.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD shows "Sensor Error!" on boot | BMP180 not wired or bad address | Verify that the BMP180 VCC is on 3.3V, GND on GND, SDA on A4, and SCL on A5. |
| Screen displays garbage characters | Signal lines crossed | Double check that SDA is wired to A4 and SCL is wired to A5 on both devices. |

## Mode Notes
These patterns (dual I2C device operations) are supported by MbedO interpreted mode.

## Related Projects
- [51 - Temp Display LCD](51-temp-display-lcd.md)
- [59 - Pressure Serial](59-pressure-serial.md)
- [60 - BMP180 Temperature Serial](60-bmp-180-temperature-serial.md)
