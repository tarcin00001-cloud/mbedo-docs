# 86 - Weather Station

Combine a BMP180 barometric pressure sensor and a DHT22 temperature/humidity sensor to build a multi-reading weather station that displays all readings on an I2C LCD.

## Goal
Learn how to integrate two I2C and one single-wire sensor into one sketch, read their data in sequence, and display formatted multi-line output on a 16x2 LCD.

## What You Will Build
The LCD shows a two-line rolling display:
- **Line 1**: Temperature (from DHT22) and Humidity percentage
- **Line 2**: Barometric pressure (from BMP180) in hPa

All readings refresh every 2 seconds.

**Why these pins?** SDA/SCL are the Arduino Uno's hardware I2C bus (A4/A5). The BMP180 and LCD share this same I2C bus at different addresses. The DHT22 uses a single-wire protocol on D2.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (DHT pull-up) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Single-wire data |
| DHT22 Sensor | GND | GND | Ground reference |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Data bus |
| BMP180 Sensor | SCL | A5 | I2C Clock bus |
| BMP180 Sensor | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus (shared) |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus (shared) |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <Adafruit_BMP085.h>
#include <DHT.h>

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;

DHT dht(DHT_PIN, DHT_TYPE);
Adafruit_BMP085 bmp;
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  dht.begin();
  bmp.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.print("Weather Station");
  delay(1500);
  lcd.clear();
  
  Serial.println("Weather Station Active");
}

void loop() {
  float humidity = dht.readHumidity();
  float tempDHT  = dht.readTemperature();
  float pressure = bmp.readPressure() / 100.0; // Convert Pa to hPa
  
  if (isnan(humidity) || isnan(tempDHT)) {
    Serial.println("Error: DHT22 read failure!");
    return;
  }
  
  Serial.print("Temp: ");    Serial.print(tempDHT);   Serial.print(" C | ");
  Serial.print("Hum: ");     Serial.print(humidity);  Serial.print("% | ");
  Serial.print("Press: ");   Serial.print(pressure);  Serial.println(" hPa");
  
  // Line 1: Temperature and Humidity
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(tempDHT, 0);
  lcd.print("C H:");
  lcd.print(humidity, 0);
  lcd.print("%   "); // Trailing spaces to clear old digits
  
  // Line 2: Barometric Pressure
  lcd.setCursor(0, 1);
  lcd.print("P:");
  lcd.print(pressure, 0);
  lcd.print("hPa     ");
  
  delay(2000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **BMP180 Sensor**, and **16x2 I2C LCD** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **SDA** to **D2**, **GND** to **GND**.
3. Connect BMP180: **VCC** to **3.3V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
4. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Double-click each sensor to adjust sliders and watch the LCD update.

## Expected Output

Terminal:
```
Weather Station Active
Temp: 24.00 C | Hum: 60.00% | Press: 1013 hPa
```

LCD Display:
```
T:24C H:60%
P:1013hPa
```

## Expected Canvas Behavior

| DHT22 Temp | DHT22 Humidity | BMP180 Pressure | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- |
| 24 C | 60% | 1013 hPa | `T:24C H:60%` | `P:1013hPa` |
| 35 C | 80% | 980 hPa | `T:35C H:80%` | `P:980hPa` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `bmp.readPressure() / 100.0` | Converts raw pressure value from Pascals to hectopascals (hPa), the standard weather unit. |
| `lcd.setCursor(0, 1)` | Positions the cursor at column 0, row 1 (second line) before printing. |
| Trailing `"   "` spaces | Clears any stale characters left over from a longer previous print (e.g., 4-digit to 3-digit transition). |

## Hardware & Safety Concept: Shared I2C Bus
I2C is a **multi-device bus** — multiple sensors can share the same two wires (SDA and SCL) as long as each device uses a unique 7-bit address.
- BMP180 always uses address `0x77`.
- Common LCD I2C backpack modules use address `0x27` or `0x3F`.
- The Arduino acts as the **bus master** and addresses each device separately in sequence.
This allows you to connect many sensors using only 2 pins, which is especially powerful on the Arduino Uno where pins are scarce.

## Try This! (Challenges)
1. **Scrolling Data**: Add a third sensor reading to the display using `lcd.scrollDisplayLeft()` to create a marquee ticker across line 2.
2. **Trend Arrow**: Compare the current pressure reading to the last saved value. If pressure dropped more than 5 hPa, print a down-arrow character on the LCD (`lcd.write(byte(0))` using a custom glyph).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD stays blank | Wrong I2C address | Try changing `0x27` to `0x3F` in the constructor. |
| BMP180 shows 0 or garbage | Missing `Wire.begin()` call | Verify that `bmp.begin()` is called in `setup()`. It internally calls `Wire.begin()`. |
| DHT22 always returns NAN | Timing issue on startup | Add a 500 ms `delay(500)` after `dht.begin()` to allow the sensor to warm up. |

## Mode Notes
These patterns (I2C multi-device bus reads and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [59 - Pressure Serial](../intermediate/59-pressure-serial.md)
- [47 - Temp Humidity Serial](../intermediate/47-temp-humidity-serial.md)
- [61 - Weather Readout LCD](../intermediate/61-weather-readout-lcd.md)
- [87 - Air Quality Monitor](87-air-quality-monitor.md)
