# 112 - ESP32 DS3231 Real Time Clock Display

Build a digital clock station that reads time and date from a DS3231 high-precision Real Time Clock (RTC) and displays it on a 16x2 I2C LCD screen.

## Goal
Learn how to read date/time structures from a dedicated battery-backed I2C RTC, initialize parameters using the `RTClib` library, and format date-time string outputs.

## What You Will Build
A DS3231 RTC and 16x2 I2C LCD are connected to the I2C bus (GPIO 21/22). The ESP32 queries the RTC every 500 ms, formats the data, and displays the current time (HH:MM:SS) on Row 1 and the date (YYYY/MM/DD) on Row 2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| DS3231 Real Time Clock Module | `rtc_ds3231` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| DS3231 Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| DS3231 Module | SDA | GPIO21 | Blue | I2C Data (shared) |
| DS3231 Module | SCL | GPIO22 | Yellow | I2C Clock (shared) |
| I2C LCD | VCC | 5V (Vin) | Red | LCD power |
| I2C LCD | GND | GND | Black | Ground reference |
| I2C LCD | SDA | GPIO21 | Blue | I2C Data (shared) |
| I2C LCD | SCL | GPIO22 | Yellow | I2C Clock (shared) |

> **Wiring tip:** Share the I2C bus pins in parallel (GPIO 21 and 22) for both the RTC and the LCD.

## Code
```cpp
// DS3231 Real Time Clock Display
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <RTClib.h>

RTC_DS3231 rtc;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(115200);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("RTC Clock Setup");
  
  if (!rtc.begin()) {
    lcd.setCursor(0, 1);
    lcd.print("RTC Error!     ");
    while (1) {}
  }
  
  // Checks if the RTC lost power (e.g. battery removed/depleted)
  // If so, sets the RTC to the date & time this sketch was compiled
  if (rtc.lostPower()) {
    Serial.println("RTC lost power, setting time...");
    rtc.adjust(DateTime(F(__DATE__), F(__TIME__)));
  }
  
  delay(1000);
  lcd.clear();
}

void loop() {
  DateTime now = rtc.now();
  
  // Format Time: HH:MM:SS
  char timeBuffer[16];
  sprintf(timeBuffer, "Time: %02d:%02d:%02d", now.hour(), now.minute(), now.second());
  
  // Format Date: YYYY/MM/DD
  char dateBuffer[16];
  sprintf(dateBuffer, "Date: %04d/%02d/%02d", now.year(), now.month(), now.day());
  
  // Print to LCD
  lcd.setCursor(0, 0);
  lcd.print(timeBuffer);
  
  lcd.setCursor(0, 1);
  lcd.print(dateBuffer);
  
  // Log to serial monitor
  Serial.print(dateBuffer); Serial.print(" | "); Serial.println(timeBuffer);
  
  delay(500); // 2Hz refresh rate
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **DS3231 RTC**, and **16x2 I2C LCD** onto the canvas.
2. Wire SDA to **GPIO21** and SCL to **GPIO22** for both components.
3. Paste the code and click **Run**.
4. Observe the current simulated system date and time counting up on the LCD.

## Expected Output
Serial Monitor:
```
Date: 2026/07/11 | Time: 00:50:00
Date: 2026/07/11 | Time: 00:50:01
```

LCD Display:
```
Time: 00:50:01
Date: 2026/07/11
```

## Expected Canvas Behavior
* The LCD displays the live time and date, updating the seconds counter every half-second.
* Standby currents of the RTC are handled automatically on the simulated chip.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `rtc.lostPower()` | Returns true if the backup battery voltage drops, indicating memory was lost. |
| `rtc.adjust(...)` | Synchronizes the RTC to the computer clock compilation time stamps (`__DATE__`, `__TIME__`). |
| `sprintf(..., "%02d")` | Pads single digits with a leading zero (e.g. "9" becomes "09") to keep the clock aligned. |

## Hardware & Safety Concept: Real Time Clocks and Temperature Compensation
Unlike microcontrollers that lose their internal clock time when powered off, a dedicated RTC contains a backup coin-cell battery (CR2032) that keeps its low-frequency 32.768 kHz oscillator running for years without main power. The DS3231 is temperature-compensated (TCXO), meaning it contains an internal temperature sensor that adjusts the oscillator frequency to prevent drift caused by ambient hot/cold temperature shifts, keeping accuracy to within \(\pm 2\) minutes per year.

## Try This! (Challenges)
1. **12-Hour format switch**: Add a slide switch on GPIO 4 that toggles the time format between 24-hour mode and 12-hour AM/PM mode.
2. **Weekly Day Indicator**: Display the day of the week (Monday, Tuesday, etc.) on the LCD using `now.dayOfTheWeek()`.
3. **Clock Temp HUD**: The DS3231 has a built-in temperature sensor. Read and print it on the screen using `rtc.getTemperature()`.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Time is incorrect or static | Battery dead or `lostPower()` adjust block skipped | Replace the CR2032 coin-cell battery; check if `rtc.adjust()` successfully triggered |
| LCD shows zero date values | Communication lost | Verify SDA and SCL pin connections |
| Time jumps when power is cycled | Reset line floating | Verify DS3231 GND is tied securely to the ESP32 GND |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [113 - ESP32 DS3231 RTC Alarm System](113-esp32-ds3231-rtc-alarm-system.md)
- [58 - ESP32 16×2 I2C LCD Print Text](58-esp32-16x2-i2c-lcd-print-text.md)
- [119 - ESP32 OLED Digital Stop Watch](119-esp32-oled-digital-stop-watch.md)
