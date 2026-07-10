# 53 - Dual Sensor LCD

Create a multi-sensor station displaying temperature from a DHT22 and light levels from an LDR on an I2C LCD screen.

## Goal
Learn how to read multiple digital and analog inputs simultaneously and organize their data layout on a multi-line LCD screen.

## What You Will Build
The Arduino reads environmental temperature from the DHT22 and ambient light levels from the LDR. The LCD display writes:
- Line 1: "T: 24.5 C" (Temperature)
- Line 2: "Light: 50%" (Scaled light percentage)
The screen updates every 1.5 seconds.

**Why D2, A0, A4, and A5?** Pin D2 reads the digital DHT22 sensor. Pin A0 reads the analog LDR light voltage. Pins A4/A5 control the I2C LCD display.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| LDR Light Sensor | `ldr` | Yes | Yes |
| LCD I2C 16x2 | `lcd_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (pull-up for DHT & LDR divider) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| LDR Sensor | VCC | 5V | Power supply (5V) |
| LDR Sensor | AO | A0 | Analog signal connection |
| LDR Sensor | GND | GND | Ground reference |
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
const int LDR_PIN = A0;

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  dht.begin();
  
  lcd.init();
  lcd.backlight();
  lcd.clear();
  
  lcd.setCursor(0, 0);
  lcd.print("Weather Station");
  lcd.setCursor(0, 1);
  lcd.print("Starting...");
  delay(1500);
}

void loop() {
  float tempC = dht.readTemperature();
  int lightRaw = analogRead(LDR_PIN);
  
  // Convert light reading (0-1023) to percentage (0-100%)
  int lightPercent = map(lightRaw, 0, 1023, 0, 100);
  
  lcd.clear();
  
  if (isnan(tempC)) {
    lcd.setCursor(0, 0);
    lcd.print("DHT Sensor Error!");
  } else {
    // Row 0: Temperature
    lcd.setCursor(0, 0);
    lcd.print("Temp:  ");
    lcd.print(tempC);
    lcd.print(" C");
    
    // Row 1: Light Percentage
    lcd.setCursor(0, 1);
    lcd.print("Light: ");
    lcd.print(lightPercent);
    lcd.print(" %");
  }
  
  delay(1500); // 1.5s refresh delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **LDR Light Sensor**, and **LCD I2C 16x2** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect LDR **VCC** to Arduino **5V**, **AO** to Arduino **A0**, and **GND** to Arduino **GND**.
4. Connect LCD **VCC** to Arduino **5V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.
8. Open the **Terminal** tab in the bottom dock.
9. Adjust the temperature slider on the DHT22 and the light slider on the LDR, and watch the values display on the LCD.

## Expected Output
The LCD screen component on the canvas updates:
```
Temp:  24.50 C
Light: 50 %
```
Values update dynamically as you move the sliders on both components.

## Expected Canvas Behavior

| Temperature Input | LDR Slider State | LCD Row 0 Display | LCD Row 1 Display |
| --- | --- | --- | --- |
| 20.5 C | 80% Light | "Temp:  20.50 C" | "Light: 80 %" |
| 35.0 C | 20% Light | "Temp:  35.00 C" | "Light: 20 %" |

The display updates automatically every 1.5 seconds.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int lightPercent = map(lightRaw, 0, 1023, 0, 100)` | Converts the 10-bit analog LDR reading into a human-friendly percentage rotation. |
| `lcd.print(tempC)` | Displays the decimal float reading directly on the active LCD segment. |

## Hardware & Safety Concept: Multi-Sensor Routing
When wiring multiple sensors, power and ground lines can quickly crowd the Arduino board pins.
- The Arduino Uno has only one 5V power socket and three GND sockets.
- In physical systems, we use a **breadboard** to distribute power: wire one 5V line and one GND line to the breadboard rails, and then route VCC and GND for all sensors and displays to those shared rails.

## Try This! (Challenges)
1. **Dynamic Alert Message**: If the light level is below 20% and temperature is below 15C, write a warning "Cold & Dark!" on the LCD.
2. **Fahrenheit switch**: Change the code to display temperature in Fahrenheit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| "DHT Sensor Error" appears | Data wire not connected | Check that the DHT22 SDA pin connects to Arduino pin D2. |
| LCD shows garbled blocks | Signal wires crossed | Ensure SDA goes to A4 and SCL goes to A5. Double check connections. |

## Mode Notes
These patterns (reading multiple analog/digital sensors and outputting to an I2C display) are supported by MbedO interpreted mode.

## Related Projects
- [18 - Light Meter](../beginner/18-light-meter.md)
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [51 - Temp Display LCD](51-temp-display-lcd.md)
