# 112 - DS3231 Real Time Clock Display

Build a battery-backed digital clock displaying precise calendar dates and time readouts on a character LCD using a DS3231 Real Time Clock (RTC) module and the VEGA ARIES v3 board.

## Goal
Learn how to interface I2C real-time clock sensors, manage calendar datetime variables, and format double-digit numbers on an LCD screen.

## What You Will Build
A standalone digital clock. The ARIES board queries time information from the DS3231 RTC module over the I2C bus every second. The current date (YYYY-MM-DD) is displayed on the top row of the 16x2 LCD, and the current time (HH:MM:SS) is displayed on the bottom row.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| DS3231 Real Time Clock | `ds3231` | Yes | Yes |
| I2C 16x2 LCD Display | `lcd` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 RTC | VCC | 3V3 | Red | RTC module power (3.3V) |
| DS3231 RTC | GND | GND | Black | Ground reference |
| DS3231 RTC | SDA | SDA0 (GP17) | Blue | I2C Data line |
| DS3231 RTC | SCL | SCL0 (GP16) | Yellow | I2C Clock line |
| I2C LCD | VCC | 5V | Red | LCD power supply (5V) |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | SDA0 (GP17) | Blue | Shared I2C Data line |
| I2C LCD | SCL | SCL0 (GP16) | Yellow | Shared I2C Clock line |

> **Wiring tip:** Both the DS3231 RTC and the I2C LCD share the same I2C bus lines (SDA0/SCL0). Connect SDA pins and SCL pins together on a breadboard. The ARIES I2C bus can address multiple devices on the same wires.

## Code
```cpp
#include <Wire.h>
#include <RTClib.h>
#include <LiquidCrystal_I2C.h>

RTC_DS3231 rtc;
LiquidCrystal_I2C lcd(0x27, 16, 2);

int lastSecond = -1;

void setup() {
  Wire.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("RTC Initialize");

  if (!rtc.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("RTC Failed!    ");
  } else {
    // If the RTC loses power, set it to the compile date and time
    if (rtc.lostPower()) {
      rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
    }
    lcd.clear();
  }
}

void loop() {
  DateTime now = rtc.now();

  // Redraw LCD only when the second changes to eliminate flicker
  if (now.second() != lastSecond) {
    // Top Row: Date (YYYY-MM-DD)
    lcd.setCursor(0, 0);
    lcd.print("Date: ");
    lcd.print(now.year());
    lcd.print("-");
    
    // Pad single digit months with a leading zero
    if (now.month() < 10) {
      lcd.print("0");
    }
    lcd.print(now.month());
    lcd.print("-");
    
    // Pad single digit days with a leading zero
    if (now.day() < 10) {
      lcd.print("0");
    }
    lcd.print(now.day());

    // Bottom Row: Time (HH:MM:SS)
    lcd.setCursor(0, 1);
    lcd.print("Time: ");
    
    // Pad hours
    if (now.hour() < 10) {
      lcd.print("0");
    }
    lcd.print(now.hour());
    lcd.print(":");
    
    // Pad minutes
    if (now.minute() < 10) {
      lcd.print("0");
    }
    lcd.print(now.minute());
    lcd.print(":");
    
    // Pad seconds
    if (now.second() < 10) {
      lcd.print("0");
    }
    lcd.print(now.second());

    lastSecond = now.second();
  }

  delay(200); // Polling delay
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **DS3231 Real Time Clock**, and **I2C LCD Display** components onto the canvas.
2. Wire the DS3231: **VCC** to **3V3**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
3. Wire the LCD: **VCC** to **5V**, **GND** to **GND**, **SDA** to **SDA0 (GP17)**, and **SCL** to **SCL0 (GP16)**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
DS3231 RTC Initialized.
Time Set to: 2026-07-11 10:37:35
```

## Expected Canvas Behavior
* On boot, the LCD clears and starts counting up time in seconds on row 2, while row 1 shows the static current date.
* The clock time tracks the system time of your simulator exactly.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `RTC_DS3231 rtc` | Declares the RTC class object using I2C communications. |
| `rtc.lostPower()` | Checks if the backup battery was disconnected, resetting clock values. |
| `rtc.adjust(...)` | Synchronizes the DS3231 to the exact date and time the code was compiled. |
| `rtc.now()` | Queries the current time structure from the DS3231 registers. |
| `now.second() != lastSecond` | Ensures the screen updates exactly once per second. |

## Hardware & Safety Concept
* **Clock Drift and Temperature Compensation**: The DS3231 is an extremely accurate, temperature-compensated real-time clock. It contains an internal temperature sensor that adjusts the frequency of its crystal oscillator. This compensates for frequency shifts caused by temperature swings, limiting clock drift to under 2 minutes per year.
* **Coin Cell Battery Life**: A CR2032 lithium coin cell battery powers the DS3231's internal registers and crystal oscillator when main power is disconnected. This ensures the clock keeps accurate time for several years without external power.

## Try This! (Challenges)
1. **12-Hour AM/PM Format**: Modify the code to display the time in 12-hour AM/PM format (e.g., `Time: 01:30:15 PM`) rather than 24-hour format.
2. **Temperature Monitor Display**: The DS3231 contains a built-in temperature sensor. Read this value using `rtc.getTemperature()` and display it on the LCD screen every 5 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display time resets to default date every boot | Missing backup coin cell | Install a CR2032 cell in the physical DS3231 battery holder. |
| LCD shows `RTC Failed!` | Incorrect I2C bus wiring | Confirm both SDA and SCL are wired correctly. Check for loose connections. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [58 - 16x2 I2C LCD Hello World](58-16x2-i2c-lcd-hello-world.md)
- [113 - DS3231 RTC Alarm System](113-ds3231-rtc-alarm-system.md)
