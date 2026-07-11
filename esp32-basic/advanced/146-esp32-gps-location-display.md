# 146 - ESP32 GPS Location Display

Build a portable GPS receiver station that parses coordinates from a NEO-6M GPS module and displays latitude, longitude, altitude, and satellite counts on a cycling 16x2 I2C LCD.

## Goal
Learn how to display coordinates on LCD screens, implement automated page switching, and format float values.

## What You Will Build
A NEO-6M GPS module is connected to UART2 (RX2: GPIO 16, TX2: GPIO 17). A 16x2 LCD is connected to I2C. If no GPS fix is acquired, the LCD displays "Searching...". Once a fix is established, the LCD cycles every 4 seconds between Page 1 (Latitude and Longitude) and Page 2 (Altitude and Satellites).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| NEO-6M GPS Module | `gps_neo6m` | Yes | Yes |
| 16x2 I2C LCD Module | `lcd_i2c` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M Module | TXD / RXD | GPIO16 / GPIO17 | Green / Yellow | GPS Serial lines |
| NEO-6M Module | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| I2C LCD | SDA / SCL | GPIO21 / GPIO22 | Blue / Yellow | I2C Bus |
| I2C LCD | VCC / GND | 5V / GND | Red / Black | LCD power |

> **Wiring tip:** Connect the GPS serial and I2C lines. Make sure the LCD is powered from 5V Vin to guarantee backlight brightness.

## Code
```cpp
// GPS Location Display (Lat/Lon/Alt -> LCD)
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <TinyGPS++.h>

#define RX2_PIN 16
#define TX2_PIN 17

TinyGPSPlus gps;
LiquidCrystal_I2C lcd(0x27, 16, 2);

unsigned long lastPageChange = 0;
int lcdPage = 0;

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
  lcd.init();
  lcd.backlight();
  lcd.setCursor(0, 0);
  lcd.print("GPS Display Unit");
  lcd.setCursor(0, 1);
  lcd.print("Starting up...");
  
  delay(1500);
  lcd.clear();
}

void loop() {
  // Feed GPS parser
  while (Serial2.available() > 0) {
    gps.encode(Serial2.read());
  }
  
  unsigned long now = millis();
  
  // Cycle display pages every 4 seconds if fix is valid
  if (gps.location.isValid()) {
    if (now - lastPageChange >= 4000) {
      lcdPage = (lcdPage + 1) % 2;
      lcd.clear();
      lastPageChange = now;
    }
    
    // Page 0: Latitude & Longitude
    if (lcdPage == 0) {
      lcd.setCursor(0, 0);
      lcd.print("LAT: ");
      lcd.print(gps.location.lat(), 5);
      
      lcd.setCursor(0, 1);
      lcd.print("LON: ");
      lcd.print(gps.location.lng(), 5);
    } 
    // Page 1: Altitude & Satellite stats
    else {
      lcd.setCursor(0, 0);
      lcd.print("ALT: ");
      lcd.print(gps.altitude.meters(), 1);
      lcd.print(" m");
      
      lcd.setCursor(0, 1);
      lcd.print("SATS: ");
      lcd.print(gps.satellites.value());
      lcd.print(" active");
    }
  } 
  // No GPS lock
  else {
    lcd.setCursor(0, 0);
    lcd.print("GPS: SEARCHING  ");
    lcd.setCursor(0, 1);
    lcd.print("Satellites: ");
    lcd.print(gps.satellites.value());
    lcd.print("   "); // Clear trailing space
    
    // Slow down cycle if no fix
    delay(200);
  }
  
  delay(10);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **NEO-6M GPS**, and **16x2 I2C LCD** onto the canvas.
2. Wire the GPS to **GPIO16/GPIO17** and LCD to **GPIO21/GPIO22**.
3. Paste the code and click **Run**.
4. Adjust coordinates on the GPS widget. Watch the LCD cycle between Lat/Lon and Altitude/Satellites.

## Expected Output
Serial Monitor:
* Coordinates mapped to LCD screens.

LCD Display (Page 0):
```
LAT: 12.97160
LON: 77.59456
```

LCD Display (Page 1):
```
ALT: 920.5 m
SATS: 6 active
```

## Expected Canvas Behavior
* While searching for satellites, LCD shows "GPS: SEARCHING".
* Dragging the satellite slider on the GPS widget above 4 updates the LCD to show coordinates.
* The LCD screen cycles between Page 0 and Page 1 every 4 seconds.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `gps.location.isValid()` | Verifies that a valid coordinate set has been parsed. |
| `now - lastPageChange >= 4000` | Checks if it is time to cycle to the next screen page. |
| `gps.location.lat()` | Retrieves the floating-point latitude coordinate. |

## Hardware & Safety Concept: NMEA Parser Sentence Structure
GPS modules transmit data sentences starting with `$GPGGA` or `$GPRMC`. These contain comma-separated raw values (e.g. `$GPGGA,123519,4807.038,N,01131.000,E,1,08,0.9,545.4,M...`). The `TinyGPS++` library decodes these coordinates, correcting raw formatting (like converting minutes/seconds coordinates to decimal degrees) so they can be read as standard float variables.

## Try This! (Challenges)
1. **Interactive Toggle button**: Connect a push button on GPIO 4 to switch pages manually, disabling the automatic cycle.
2. **Speedometer Mode**: Add a third page that displays the speed in km/h or mph.
3. **Chime on satellite lock**: Sound a buzzer (GPIO 15) when the GPS shifts from searching to a valid coordinate fix.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays stuck on "GPS: SEARCHING" | No satellite signal | Move the GPS module outside or near a window; check that the satellite count slider is turned up in the simulator |
| LCD display flickers | Screen cleared too often | Ensure `lcd.clear()` is called only when the page changes, not in every loop execution |
| Coordinates display with fewer decimals | Printing without float specs | Use `lcd.print(val, 5)` to show 5 decimal places |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [145 - ESP32 GPS Logger](145-esp32-gps-logger.md)
- [147 - ESP32 GPS Velocity Logger](147-esp32-gps-velocity-logger.md)
- [58 - ESP32 16×2 I2C LCD Print Text](../intermediate/58-esp32-16x2-i2c-lcd-print-text.md)
