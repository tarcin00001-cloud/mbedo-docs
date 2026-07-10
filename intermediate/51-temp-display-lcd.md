# 51 - Temp Display LCD

Display real-time temperature and humidity readings from a DHT22 sensor on a 16x2 I2C LCD screen.

## Goal
Learn how to initialize and control an I2C character LCD display using the `LiquidCrystal_I2C.h` library, showing multi-line formatted data dynamically.

## What You Will Build
The Arduino reads environmental data from the DHT22 sensor. Instead of printing to the Terminal, it writes the formatted readings directly to a 16x2 character screen (e.g. Line 1: "Temp: 24.5 C", Line 2: "Humid: 55.2 %") updating every 1.5 seconds.

**Why D2, A4, and A5?** Pin D2 communicates with the DHT22 sensor. Pins A4 (SDA) and A5 (SCL) comprise the I2C bus required to control the LCD.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| LCD I2C 16x2 | `lcd_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (pull-up for DHT data) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| LCD Module | VCC | 5V | Power supply (5V) |
| LCD Module | SDA | A4 | I2C Serial Data line |
| LCD Module | SCL | A5 | I2C Serial Clock line |
| LCD Module | GND | GND | Ground reference |

## Code
```cpp
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN = 2;
const int DHT_TYPE = DHT22;

DHT dht(DHT_PIN, DHT_TYPE);

// Set LCD address to 0x27 for a 16 chars and 2 line display
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  dht.begin();
  
  // Initialize the LCD display
  lcd.init();
  lcd.backlight();
  lcd.clear();
  
  // Print startup message
  lcd.setCursor(0, 0);
  lcd.print("Sensor Station");
  lcd.setCursor(0, 1);
  lcd.print("Initializing...");
  delay(1500);
}

void loop() {
  float tempC = dht.readTemperature();
  float humidity = dht.readHumidity();
  
  lcd.clear();
  
  if (isnan(tempC) || isnan(humidity)) {
    lcd.setCursor(0, 0);
    lcd.print("Sensor Error!");
  } else {
    // Print Temperature on row 0
    lcd.setCursor(0, 0);
    lcd.print("Temp: ");
    lcd.print(tempC);
    lcd.print(" C");
    
    // Print Humidity on row 1
    lcd.setCursor(0, 1);
    lcd.print("Humid: ");
    lcd.print(humidity);
    lcd.print(" %");
  }
  
  delay(1500); // 1.5s refresh delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **LCD I2C 16x2** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect LCD **VCC** to Arduino **5V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the DHT22 sensor, adjust the sliders, and watch the values display on the LCD screen on the canvas.

## Expected Output
The LCD screen component on the canvas lights up and displays:
```
Temp: 24.50 C
Humid: 55.20 %
```
As you move the sliders on the DHT22 sensor, the screen updates to reflect the new values.

## Expected Canvas Behavior

| Temperature Input | Humidity Input | LCD Line 1 Display | LCD Line 2 Display |
| --- | --- | --- | --- |
| 28.3 C | 42.1% | "Temp: 28.30 C" | "Humid: 42.10 %" |
| 15.0 C | 88.0% | "Temp: 15.00 C" | "Humid: 88.00 %" |

The display updates automatically every 1.5 seconds.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `#include <LiquidCrystal_I2C.h>` | Imports the I2C LCD control library. |
| `LiquidCrystal_I2C lcd(0x27, 16, 2)` | Creates an LCD driver object on I2C address `0x27` (standard for PCF8574 backpacks) with 16 columns and 2 rows. |
| `lcd.init()` | Prepares the I2C bus and resets the display. |
| `lcd.backlight()` | Switches on the LED backlight backlight. |
| `lcd.setCursor(col, row)` | Moves the character drawing cursor (0-indexed: col is 0-15, row is 0-1). |
| `lcd.print("...")` | Draws text characters onto the display starting at the current cursor position. |

## Hardware & Safety Concept: The I2C Communication Bus
The **I2C (Inter-Integrated Circuit)** bus allows the Arduino to communicate with multiple devices using only two pins: **SDA** (Serial Data) and **SCL** (Serial Clock). 
- Devices are connected in parallel on these two lines.
- Each device has a unique hexadecimal address (e.g. `0x27` for the LCD, `0x68` for an RTC). The Arduino sends the address first down the wire, and only the device matching that address responds. This allows saving dozens of pin lines compared to old parallel-bus wiring configurations.

## Try This! (Challenges)
1. **Fahrenheit Toggle**: Modify the code to display Fahrenheit on line 1 instead of Celsius.
2. **Scrolling Text**: Add a check: if the temperature exceeds 30C, display "[ALERT] OVERHEAT" on the screen.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD is blank but lit | Contrast potentiometer needs adjustment | On physical hardware, turn the contrast screw on the I2C backpack board. |
| Nothing appears on display | Address mismatch or I2C wires reversed | Verify SDA is on A4 and SCL is on A5. Check that your display address is `0x27` (some use `0x3F`). |

## Mode Notes
These patterns (DHT library calls and I2C LCD character printing) are supported by MbedO interpreted mode.

## Related Projects
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [52 - Humidity Display OLED](52-humidity-display-oled.md)
- [53 - Dual Sensor LCD](53-dual-sensor-lcd.md)
