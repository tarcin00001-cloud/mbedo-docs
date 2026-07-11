# 146 - GPS Location Display

Parse latitude, longitude, and altitude from a NEO-6M GPS module and display the live position on a 16×2 I2C LCD.

## Goal

Learn how to extract specific fields from a `$GPGGA` NMEA sentence arriving on `Serial2`, convert raw NMEA degree-minute format to decimal degrees, and refresh an I2C LCD with the parsed coordinates so the board functions as a standalone GPS position display.

## What You Will Build

A NEO-6M GPS module streams NMEA sentences to ARIES `Serial2` (GPIO 16/17). The firmware accumulates characters into a `String` buffer, detects complete `$GPGGA` sentences, extracts latitude, longitude, and altitude fields using `String::indexOf()` and `String::substring()`, converts them to decimal degrees, and writes two lines to a PCF8574-backed 16×2 I2C LCD: latitude on line 1, longitude and altitude on line 2.

## Parts Needed

| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| u-blox NEO-6M GPS Module | `neo6m` | Yes | Yes |
| 16×2 I2C LCD (PCF8574) | `lcd16x2i2c` | Yes | Yes |
| Active GPS Antenna (SMA) | — | No | Yes |
| Jumper Wires | — | No | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| NEO-6M | VCC | 3V3 | Red | 3.3 V power |
| NEO-6M | GND | GND | Black | Common ground |
| NEO-6M | TX | GPIO 17 (UART2 RX) | Green | GPS data → ARIES |
| NEO-6M | RX | GPIO 16 (UART2 TX) | Blue | ARIES → GPS (config only) |
| I2C LCD | VCC | 5V | Red | PCF8574 I2C backpack prefers 5 V |
| I2C LCD | GND | GND | Black | Common ground |
| I2C LCD | SDA | GPIO 0 (I2C0 SDA) | Orange | I2C data |
| I2C LCD | SCL | GPIO 1 (I2C0 SCL) | Yellow | I2C clock |

> **Wiring tip:** The LCD I2C backpack (PCF8574) is typically at address `0x27`; some modules use `0x3F`. If the display shows nothing, run an I2C scanner sketch to find the correct address and update `LiquidCrystal_I2C lcd(0x27, 16, 2)` accordingly. The NEO-6M TX and ARIES I2C lines are completely independent buses — they do not interfere with each other.

## Code

```cpp
// 146 - GPS Location Display
// NEO-6M NMEA parser -> Lat/Lon/Alt on I2C LCD
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

// LCD at I2C address 0x27, 16 columns, 2 rows
LiquidCrystal_I2C lcd(0x27, 16, 2);

#define GPS_BAUD    9600
#define SERIAL_BAUD 115200

// NMEA sentence accumulation (String, no char array)
String nmeaSentence = "";
int    sentenceReady = 0;

// Parsed GPS fields
float gpsLatDeg  = 0.0;
float gpsLonDeg  = 0.0;
float gpsAltM    = 0.0;
int   gpsFixOk   = 0;
int   lastFixOk  = -1;  // force first draw

// Field parsing helpers
int    commaPos  = 0;
int    nextComma = 0;
int    fieldIdx  = 0;
String fieldVal  = "";

// Raw NMEA lat/lon strings
String rawLat    = "";
String latDir    = "";
String rawLon    = "";
String lonDir    = "";
String rawAlt    = "";
String rawFix    = "";

// Conversion helpers
float rawDegMin  = 0.0;
int   degPart    = 0;
float minPart    = 0.0;

// LCD display strings
String lcdLine1  = "";
String lcdLine2  = "";

// Single incoming character
char gpsCh = 0;

// LCD refresh throttle
unsigned long lastLcdRefresh = 0;
unsigned long nowMs          = 0;

// ---- Field extraction from a comma-separated NMEA sentence ----
// Returns the Nth comma-delimited field (0-indexed) from a sentence String.
// Implemented inline in loop() using state variables — no helper function.

void setup() {
  Serial.begin(SERIAL_BAUD);
  Serial2.begin(GPS_BAUD);

  Wire.begin();
  lcd.init();
  lcd.backlight();

  lcd.setCursor(0, 0);
  lcd.print("GPS Location Disp");
  lcd.setCursor(0, 1);
  lcd.print("Waiting for fix..");

  Serial.println("=== GPS Location Display ===");
  Serial.println("Waiting for $GPGGA sentence...");
}

void loop() {
  // ---- Step 1: Accumulate one NMEA sentence ----
  sentenceReady = 0;

  if (Serial2.available() > 0) {
    gpsCh = (char)Serial2.read();

    if (gpsCh == '$') {
      // Start of a new sentence — reset accumulator
      nmeaSentence = "$";
    } else if (gpsCh == '\n') {
      // End of sentence
      sentenceReady = 1;
    } else if (gpsCh != '\r') {
      nmeaSentence += gpsCh;
    }
  }

  // ---- Step 2: Parse only $GPGGA sentences ----
  if (sentenceReady && nmeaSentence.startsWith("$GPGGA")) {

    // Extract field 2 (latitude raw, e.g. "1234.5678")
    // Extract field 3 (lat direction, "N" or "S")
    // Extract field 4 (longitude raw, e.g. "07712.3456")
    // Extract field 5 (lon direction, "E" or "W")
    // Extract field 6 (fix quality, "0"=no fix, "1"=GPS, "2"=DGPS)
    // Extract field 9 (altitude in metres)
    // Fields are indexed 0..N where 0 = "$GPGGA"

    // We iterate through the sentence string extracting comma fields.
    // State variables track position; no helper function used.

    fieldIdx = 0;
    commaPos = 0;
    nextComma = 0;
    rawLat  = "";
    latDir  = "";
    rawLon  = "";
    lonDir  = "";
    rawFix  = "";
    rawAlt  = "";

    // Find fields 1-9 using indexOf with startPos
    int pos = 0;
    int prev = 0;

    // Field 0 is "$GPGGA" itself; we need fields 1..9
    // Use a state-machine approach: advance prev to next comma each step
    prev = nmeaSentence.indexOf(',', 0);         // after field 0
    int f1s = prev + 1;
    prev = nmeaSentence.indexOf(',', f1s);       // after field 1 (time)
    int f2s = prev + 1;
    prev = nmeaSentence.indexOf(',', f2s);       // after field 2 (lat)
    rawLat = nmeaSentence.substring(f2s, prev);
    int f3s = prev + 1;
    prev = nmeaSentence.indexOf(',', f3s);       // after field 3 (N/S)
    latDir = nmeaSentence.substring(f3s, prev);
    int f4s = prev + 1;
    prev = nmeaSentence.indexOf(',', f4s);       // after field 4 (lon)
    rawLon = nmeaSentence.substring(f4s, prev);
    int f5s = prev + 1;
    prev = nmeaSentence.indexOf(',', f5s);       // after field 5 (E/W)
    lonDir = nmeaSentence.substring(f5s, prev);
    int f6s = prev + 1;
    prev = nmeaSentence.indexOf(',', f6s);       // after field 6 (fix)
    rawFix = nmeaSentence.substring(f6s, prev);
    int f7s = prev + 1;
    prev = nmeaSentence.indexOf(',', f7s);       // after field 7 (sats)
    int f8s = prev + 1;
    prev = nmeaSentence.indexOf(',', f8s);       // after field 8 (HDOP)
    int f9s = prev + 1;
    prev = nmeaSentence.indexOf(',', f9s);       // after field 9 (alt)
    if (prev == -1) prev = nmeaSentence.length();
    rawAlt = nmeaSentence.substring(f9s, prev);

    gpsFixOk = rawFix.toInt();

    if (gpsFixOk > 0 && rawLat.length() > 0 && rawLon.length() > 0) {
      // Convert NMEA lat DDMM.MMMM -> decimal degrees
      rawDegMin = rawLat.toFloat();
      degPart   = (int)(rawDegMin / 100);
      minPart   = rawDegMin - (degPart * 100.0);
      gpsLatDeg = degPart + (minPart / 60.0);
      if (latDir == "S") gpsLatDeg = -gpsLatDeg;

      // Convert NMEA lon DDDMM.MMMM -> decimal degrees
      rawDegMin = rawLon.toFloat();
      degPart   = (int)(rawDegMin / 100);
      minPart   = rawDegMin - (degPart * 100.0);
      gpsLonDeg = degPart + (minPart / 60.0);
      if (lonDir == "W") gpsLonDeg = -gpsLonDeg;

      gpsAltM = rawAlt.toFloat();

      Serial.print("Lat: "); Serial.print(gpsLatDeg, 5);
      Serial.print("  Lon: "); Serial.print(gpsLonDeg, 5);
      Serial.print("  Alt: "); Serial.print(gpsAltM, 1);
      Serial.println(" m");
    }
  }

  // ---- Step 3: Refresh LCD at most once per second ----
  nowMs = millis();
  if (nowMs - lastLcdRefresh >= 1000) {
    lastLcdRefresh = nowMs;

    if (gpsFixOk == 0) {
      lcd.setCursor(0, 0);
      lcd.print("No GPS Fix      ");
      lcd.setCursor(0, 1);
      lcd.print("Searching...    ");
    } else {
      // Line 1: Lat with direction indicator
      lcdLine1 = String(gpsLatDeg < 0 ? "S" : "N");
      lcdLine1 += String(abs(gpsLatDeg), 4);
      while (lcdLine1.length() < 16) lcdLine1 += " ";
      lcd.setCursor(0, 0);
      lcd.print(lcdLine1.substring(0, 16));

      // Line 2: Lon + alt abbreviated
      lcdLine2 = String(gpsLonDeg < 0 ? "W" : "E");
      lcdLine2 += String(abs(gpsLonDeg), 4);
      lcdLine2 += " ";
      lcdLine2 += String((int)gpsAltM);
      lcdLine2 += "m";
      while (lcdLine2.length() < 16) lcdLine2 += " ";
      lcd.setCursor(0, 1);
      lcd.print(lcdLine2.substring(0, 16));
    }
  }
}
```

## What to Click in MbedO

1. Drag **VEGA ARIES v3**, **NEO-6M GPS**, and **16×2 I2C LCD** onto the canvas.
2. Wire the NEO-6M: **TX → GPIO 17**, **RX → GPIO 16**, **VCC → 3V3**, **GND → GND**.
3. Wire the LCD: **SDA → GPIO 0**, **SCL → GPIO 1**, **VCC → 5V**, **GND → GND**.
4. Paste the code into the editor and select **Interpreted Mode**.
5. Click **Run**.
6. In the MbedO GPS widget, adjust latitude, longitude, and altitude sliders to simulate position. The LCD updates within one second.

## Expected Output

Serial Monitor:
```
=== GPS Location Display ===
Waiting for $GPGGA sentence...
Lat: 12.57613  Lon: 77.20576  Alt: 216.5 m
Lat: 12.57614  Lon: 77.20577  Alt: 216.5 m
```

LCD Display:
```
Line 1: N12.5761
Line 2: E77.2058 216m
```

## Expected Canvas Behavior

* Before a GPS fix: LCD shows `No GPS Fix` / `Searching...`.
* After fix acquired: Line 1 shows latitude with N/S prefix; line 2 shows longitude with E/W prefix and altitude in metres.
* The LCD refreshes once per second regardless of GPS update rate.
* The Serial Monitor logs every parsed fix for debugging.

## Code Walkthrough

| Line | Check / Action |
| --- | --- |
| `nmeaSentence.startsWith("$GPGGA")` | Filters only the GGA sentence — the one that carries Lat, Lon, fix quality, and altitude. |
| `nmeaSentence.indexOf(',', pos)` | Walks through comma-delimited fields without splitting into an array. |
| `nmeaSentence.substring(f2s, prev)` | Extracts the raw latitude field between its surrounding commas. |
| `rawDegMin / 100` | NMEA encodes latitude as `DDMM.MMMM`; integer-dividing by 100 gives the degree component. |
| `degPart + (minPart / 60.0)` | Converts minutes to fractional degrees: decimal degrees = degrees + minutes ÷ 60. |
| `gpsFixOk > 0` | Field 6 of GPGGA: `0` = no fix, `1` = GPS fix, `2` = DGPS. Only update display when > 0. |
| `lcd.print(lcdLine1.substring(0, 16))` | Truncates output to exactly 16 characters to prevent cursor wrap on the LCD. |
| `nowMs - lastLcdRefresh >= 1000` | Throttles LCD writes to 1 Hz, avoiding I2C bus saturation and display flicker. |

## Hardware & Safety Concept

* **NMEA 0183 Field Indexing**: NMEA sentences are comma-separated ASCII strings. The `$GPGGA` sentence contains 15 fields indexed from 0. Field 2 is raw latitude in `DDMM.MMMM` format, not decimal degrees. The conversion `D + M/60` is mandatory — skipping it would place coordinates hundreds of kilometres off. Longitude uses `DDDMM.MMMM` (three degree digits).
* **LCD Throttling**: I2C at 100 kHz can sustain many LCD refreshes per second, but unnecessary writes cause visible flicker and waste CPU cycles. A 1-second refresh interval matches the GPS 1 Hz update rate and provides a smooth user experience.
* **Direction Signs**: NMEA always outputs positive values — direction (N/S, E/W) is in the adjacent field. Southern latitudes and western longitudes must be negated after conversion to standard signed decimal-degrees convention.

## Try This! (Challenges)

1. **Satellite Count Display**: Extract field 7 of `$GPGGA` (number of satellites in use) and append it to LCD line 2, e.g. `E77.2058 8sat`.
2. **Fix Quality Indicator**: Use field 6 to show a `*` on LCD line 1 when fix quality is 2 (DGPS — more accurate), so the user can distinguish between standard and differential GPS accuracy.

## Troubleshooting

| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays blank | Wrong I2C address | Run an I2C scanner; change `0x27` to `0x3F` or whichever address is found. |
| `No GPS Fix` persists indoors | Weak satellite signal | Move antenna near a window or outdoors; wait up to 5 minutes for cold-start fix. |
| Coordinates look wildly wrong | NMEA format not converted | Ensure the DDMM→decimal conversion (`degPart + minPart/60`) is applied — raw NMEA values are not decimal degrees. |
| LCD shows garbled characters | Power issue | Ensure LCD VCC is connected to 5 V, not 3V3; contrast potentiometer on backpack may need adjustment. |

## Mode Notes

This project runs in MbedO **interpreted C++ mode**.

## Related Projects

- [145 - GPS Logger](145-gps-logger.md)
- [147 - GPS Velocity Logger](147-gps-velocity-logger.md)
- [148 - I2C OLED Digital Compass](148-i2c-oled-digital-compass.md)
