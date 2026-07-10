# 81 - Temp Number TM1637

Display real-time temperature readings from a DHT22 sensor on a 4-digit TM1637 7-segment display.

## Goal
Learn how to read temperature float data from a DHT22 sensor, convert the float to a whole integer, and display it on a TM1637 screen.

## What You Will Build
The Arduino reads environmental temperature from the DHT22, casts the decimal float to a whole number, and displays the temperature (e.g. `24`) on the 7-segment display screen, refreshing every 1.5 seconds.

**Why D2, D3, and D4?** Pins D2/D3 manage the display. Pin D4 communicates with the DHT22 sensor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| TM1637 Display | `seven_segment_tm1637` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (pull-up for DHT data) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D4 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| TM1637 Display | CLK | D2 | Serial Clock line |
| TM1637 Display | DIO | D3 | Serial Data line |
| TM1637 Display | VCC | 5V | Power supply (5V) |
| TM1637 Display | GND | GND | Ground reference |

## Code
```cpp
#include <DHT.h>
#include <TM1637Display.h>

const int CLK_PIN  = 2;
const int DIO_PIN  = 3;
const int DHT_PIN  = 4;
const int DHT_TYPE = DHT22;

TM1637Display display(CLK_PIN, DIO_PIN);
DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  dht.begin();
  
  display.setBrightness(5);
  display.clear();
  
  Serial.begin(9600);
  Serial.println("DHT22 TM1637 Station Active");
}

void loop() {
  float tempC = dht.readTemperature();
  
  if (isnan(tempC)) {
    Serial.println("Error: Failed to read from DHT22 sensor!");
    display.clear();
  } else {
    // Cast float (e.g. 24.50) to integer (24) for 7-segment display
    int tempInt = (int)tempC;
    
    Serial.print("Temperature: ");
    Serial.print(tempC);
    Serial.println(" C");
    
    // Display value on TM1637
    display.showNumberDec(tempInt);
  }
  
  delay(1500); // 1.5s refresh rate
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **TM1637 7-Segment Display** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **SDA** to **D4**, and **GND** to **GND**.
3. Connect TM1637: **CLK** to **D2**, **DIO** to **D3**, **VCC** to **5V**, and **GND** to **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Double-click the DHT22 sensor, adjust its temperature slider, and watch the values display on the 7-segment screen on the canvas.

## Expected Output
The TM1637 display component on the canvas lights up and displays:
```
  24
```
The number shown tracks the temperature slider value of the DHT22.

## Expected Canvas Behavior

| DHT22 Temp Slider Input | Float reading | TM1637 Integer display |
| --- | --- | --- |
| 20.0 C | 20.00 | `  20` |
| 35.5 C | 35.50 | `  35` |
| -5.2 C | -5.20 | `  -5` |

The display updates automatically every 1.5 seconds.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int tempInt = (int)tempC;` | Performs type casting, truncating the decimal portion of the float reading so it can be display as a whole number. |
| `display.showNumberDec(tempInt)` | Writes the integer value onto the display. |

## Hardware & Safety Concept: Displaying Decimals on 7-Segment Display
A basic TM1637 display has no dedicated hardware floating-point layout; it can only show whole numbers.
- If you need to display decimals (e.g. `24.5`), you must use special formatting techniques.
- You can multiply the temperature by 10 (e.g. `245`) and display it with a decimal point activated at the second position (`display.showNumberDecEx(245, 0b01000000)`), making it read as `24.5` on the 4-digit screen.

## Try This! (Challenges)
1. **Fahrenheit Display**: Change the code to read and display the temperature in Fahrenheit.
2. **Decimal Hack**: Modify the code to display one decimal place (e.g. `245` representing `24.5` C) using the decimal point override.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Display is completely blank | Sensor offline or read failure | Check the Terminal to confirm the raw sensor reading is not returning `NAN`. Verify SDA is on pin D4. |
| Numbers display backwards | Segment wiring swapped | Check that CLK connects to D2 and DIO connects to D3. |

## Mode Notes
These patterns (DHT sensor reads, type-casting, and TM1637 display updates) are supported by MbedO interpreted mode.

## Related Projects
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [51 - Temp Display LCD](51-temp-display-lcd.md)
- [78 - Potentiometer Number](78-potentiometer-number.md)
