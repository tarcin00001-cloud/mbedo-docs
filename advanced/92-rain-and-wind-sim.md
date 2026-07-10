# 92 - Rain and Wind Sim

Simulate a rain gauge and wind anemometer using two potentiometers, display reading levels on an LCD, and drive a LED bar graph to show wind intensity visually.

## Goal
Learn how to interpret dual analog inputs as independent physical measurement proxies, display scaled engineering units (mm/hr rain, km/h wind), and drive a proportional LED bar graph output.

## What You Will Build
Two potentiometers simulate sensor dials:
- **Pot 1 (A0)**: Rain gauge (0 to 100 mm/hr)
- **Pot 2 (A1)**: Wind anemometer (0 to 120 km/h)

The LCD shows both readings. A 4-LED bar graph illuminates proportionally to wind intensity (0, 1, 2, 3, or 4 LEDs lit based on speed band).

**Why A0, A1, D3-D6?** Pins A0/A1 read the two potentiometers. Pins D3 to D6 drive four bar graph LEDs.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Potentiometer (x2) | `potentiometer` | Yes (x2) | Yes (x2) |
| LED (x4) | `led` | Yes (x4) | Yes (x4) |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes (per LED) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Potentiometer 1 (Rain) | 1 | 5V | Positive reference |
| Potentiometer 1 (Rain) | 2 (Wiper) | A0 | Analog signal |
| Potentiometer 1 (Rain) | 3 | GND | Ground reference |
| Potentiometer 2 (Wind) | 1 | 5V | Positive reference |
| Potentiometer 2 (Wind) | 2 (Wiper) | A1 | Analog signal |
| Potentiometer 2 (Wind) | 3 | GND | Ground reference |
| LED 1 (Calm) | Anode (+) | D3 | Lowest wind indicator |
| LED 1 | Cathode (-) | GND | Via 220 ohm resistor |
| LED 2 (Breezy) | Anode (+) | D4 | Second wind indicator |
| LED 2 | Cathode (-) | GND | Via 220 ohm resistor |
| LED 3 (Windy) | Anode (+) | D5 | Third wind indicator |
| LED 3 | Cathode (-) | GND | Via 220 ohm resistor |
| LED 4 (Stormy) | Anode (+) | D6 | Highest wind indicator |
| LED 4 | Cathode (-) | GND | Via 220 ohm resistor |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int RAIN_PIN = A0;
const int WIND_PIN = A1;

// LED bar graph pins (low -> high wind intensity)
const int LED_PINS[] = {3, 4, 5, 6};
const int LED_COUNT   = 4;

LiquidCrystal_I2C lcd(0x27, 16, 2);

void updateLEDBar(int activeLEDs) {
  for (int i = 0; i < LED_COUNT; i++) {
    digitalWrite(LED_PINS[i], i < activeLEDs ? HIGH : LOW);
  }
}

void setup() {
  Serial.begin(9600);
  
  for (int i = 0; i < LED_COUNT; i++) {
    pinMode(LED_PINS[i], OUTPUT);
    digitalWrite(LED_PINS[i], LOW);
  }
  
  lcd.init();
  lcd.backlight();
  lcd.print("Rain & Wind Sim");
  delay(1500);
  lcd.clear();
  
  Serial.println("Rain and Wind Station Active");
}

void loop() {
  int rainRaw = analogRead(RAIN_PIN);
  int windRaw = analogRead(WIND_PIN);
  
  // Scale 0-1023 ADC range to engineering units
  int rainMM = map(rainRaw, 0, 1023, 0, 100);  // 0-100 mm/hr
  int windKMH = map(windRaw, 0, 1023, 0, 120); // 0-120 km/h
  
  Serial.print("Rain: "); Serial.print(rainMM);
  Serial.print(" mm/hr | Wind: "); Serial.print(windKMH);
  Serial.println(" km/h");
  
  // Update LCD
  lcd.setCursor(0, 0);
  lcd.print("Rain:");
  lcd.print(rainMM);
  lcd.print("mm/hr   ");
  
  lcd.setCursor(0, 1);
  lcd.print("Wind:");
  lcd.print(windKMH);
  lcd.print("km/h   ");
  
  // Determine LED bar level based on wind speed
  int activeLEDs;
  if (windKMH < 20) {
    activeLEDs = 0; // Calm
  } else if (windKMH < 50) {
    activeLEDs = 1; // Light Breeze
  } else if (windKMH < 80) {
    activeLEDs = 2; // Breezy
  } else if (windKMH < 100) {
    activeLEDs = 3; // Windy
  } else {
    activeLEDs = 4; // Stormy
  }
  
  updateLEDBar(activeLEDs);
  
  delay(300); // ~3 Hz refresh
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **two Potentiometers**, **four LEDs**, and **16x2 I2C LCD** onto the canvas.
2. Connect Potentiometer 1 (Rain): wiper to **A0**, outer pins to **5V** and **GND**.
3. Connect Potentiometer 2 (Wind): wiper to **A1**, outer pins to **5V** and **GND**.
4. Connect LEDs: anodes to **D3**, **D4**, **D5**, **D6** and cathodes to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Adjust the two potentiometer dials and watch the LCD values and LED bar graph respond.

## Expected Output

Terminal:
```
Rain and Wind Station Active
Rain: 35 mm/hr | Wind: 72 km/h
```

LCD Display:
```
Rain:35mm/hr
Wind:72km/h
```

LED Bar:
```
LED D3: ON
LED D4: ON
LED D5: ON
LED D6: OFF
```
(3 of 4 LEDs lit for wind 72 km/h in the "Windy" band.)

## Expected Canvas Behavior

| Wind Pot Value | Scaled Speed | LEDs Active | Wind Category |
| --- | --- | --- | --- |
| 0% | 0 km/h | 0 | Calm |
| 25% | 30 km/h | 1 | Light Breeze |
| 55% | 66 km/h | 2 | Breezy |
| 78% | 94 km/h | 3 | Windy |
| 100% | 120 km/h | 4 | Stormy |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `map(windRaw, 0, 1023, 0, 120)` | Linearly scales the raw ADC value (0-1023) to the wind speed range (0-120 km/h). |
| `updateLEDBar(activeLEDs)` | Iterates the LED array and compares each LED index against the active count, turning each LED ON or OFF accordingly. |
| `LED_PINS[]` array | Stores pin numbers in an array so the bar graph update can use a single `for` loop instead of 4 separate `if` statements. |

## Hardware & Safety Concept: Sensor Simulation with Potentiometers
Potentiometers are ideal stand-ins for resistive sensors (LDRs, thermistors, strain gauges, rain resistors) during early prototyping. They produce the same 0V to 5V analog voltage range as real sensors, allowing you to:
1. Develop and test your scaling formulas.
2. Verify your display layout and LED behavior.
3. Test edge cases (e.g. 0 km/h and 120 km/h extremes) by turning the dial.
This technique is called **hardware-in-the-loop simulation** and is widely used in embedded systems development.

## Try This! (Challenges)
1. **Beaufort Scale**: Extend the wind categories to use the standard 13-level Beaufort wind scale and display the category name (e.g. "Calm", "Moderate Breeze", "Hurricane") on the LCD.
2. **Daily Total Rain**: Accumulate the rain reading over time using `millis()`. Every 5 seconds, add `rainMM * (5 / 3600.0)` to a running total and display the daily total rain estimate on line 2.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Wrong pot controls wrong reading | Pot wiper pins mixed up | Confirm that Rain pot wiper connects to A0 and Wind pot wiper connects to A1. |
| LED bar all on or all off | LED pins not in `LED_PINS[]` array | Verify the array matches the actual wiring (D3, D4, D5, D6). |

## Mode Notes
These patterns (dual analog reads, `map()` scaling, LED array updates, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [86 - Weather Station](86-weather-station.md)
- [78 - Potentiometer Number](../intermediate/78-potentiometer-number.md)
- [83 - Motor Speed Dial](../intermediate/83-motor-speed-dial.md)
