# 100 - Smart Fan

Read temperature from a DHT22 sensor, map the value to a motor speed, drive a DC motor at that speed, and display the current temperature and fan speed on a 16x2 I2C LCD.

## Goal
Learn how to use the `map()` function to translate a sensor reading into a proportional PWM output, and how to refresh an LCD display with live data.

## What You Will Build
Every second the sketch reads temperature from the DHT22. It maps the temperature (20–40 °C range) to a PWM duty cycle (0–255). The DC motor spins proportionally — stopped when cool, full speed when hot. The LCD shows the temperature on line 1 and the calculated fan speed percentage on line 2.

**Why these pins?** The DHT22 data wire goes to `D2`. The L298N ENA pin (motor speed) goes to `D9` (a PWM-capable pin). IN1/IN2 on `D7`/`D8` set motor direction. The LCD uses the I2C bus on A4/A5.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| DC Motor | `dc_motor` | Yes | Yes |
| L298N Motor Driver | `l298n` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (DHT pull-up) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 5V | Power supply |
| DHT22 | DATA | D2 | Single-wire data |
| DHT22 | GND | GND | Ground reference |
| L298N | ENA | D9 | PWM speed control |
| L298N | IN1 | D7 | Direction pin 1 |
| L298N | IN2 | D8 | Direction pin 2 |
| L298N | VCC | 5V | Logic power |
| L298N | GND | GND | Ground reference |
| DC Motor | Terminal A | L298N OUT1 | Motor winding |
| DC Motor | Terminal B | L298N OUT2 | Motor winding |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data |
| 16x2 I2C LCD | SCL | A5 | I2C Clock |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <DHT.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;
const int ENA_PIN  = 9;
const int IN1_PIN  = 7;
const int IN2_PIN  = 8;

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  dht.begin();

  pinMode(ENA_PIN, OUTPUT);
  pinMode(IN1_PIN, OUTPUT);
  pinMode(IN2_PIN, OUTPUT);

  // Set forward direction
  digitalWrite(IN1_PIN, HIGH);
  digitalWrite(IN2_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Smart Fan Ready");
  delay(1500);
  lcd.clear();

  Serial.println("Smart Fan Ready");
}

void loop() {
  float t = dht.readTemperature();

  if (isnan(t)) {
    Serial.println("DHT22 read error");
    delay(1000);
    return;
  }

  // Map temperature 20-40 C to PWM 0-255
  int fanSpeed = map((int)t, 20, 40, 0, 255);
  if (fanSpeed < 0)   fanSpeed = 0;
  if (fanSpeed > 255) fanSpeed = 255;

  int fanPercent = map(fanSpeed, 0, 255, 0, 100);

  analogWrite(ENA_PIN, fanSpeed);

  Serial.print("Temp: ");
  Serial.print(t);
  Serial.print(" C | Fan: ");
  Serial.print(fanPercent);
  Serial.println("%");

  lcd.setCursor(0, 0);
  lcd.print("Temp: ");
  lcd.print(t, 1);
  lcd.print(" C  ");

  lcd.setCursor(0, 1);
  lcd.print("Fan:  ");
  lcd.print(fanPercent);
  lcd.print("%   ");

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **DC Motor**, **L298N Motor Driver**, and **16x2 I2C LCD** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **DATA** to **D2**, **GND** to **GND**.
3. Connect L298N: **ENA** to **D9**, **IN1** to **D7**, **IN2** to **D8**, **VCC** to **5V**, **GND** to **GND**.
4. Connect DC Motor between **L298N OUT1** and **OUT2**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Double-click the DHT22 slider and adjust temperature — watch the motor speed and LCD change.

## Expected Output

Terminal:
```
Smart Fan Ready
Temp: 25.00 C | Fan: 25%
Temp: 35.00 C | Fan: 75%
```

LCD Display:
```
Temp: 25.0 C
Fan:  25%
```

## Expected Canvas Behavior
| DHT22 Temperature | PWM Value | Fan % | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- |
| 20 °C | 0 | 0% | `Temp: 20.0 C` | `Fan:  0%` |
| 30 °C | 128 | 50% | `Temp: 30.0 C` | `Fan:  50%` |
| 40 °C | 255 | 100% | `Temp: 40.0 C` | `Fan:  100%` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `float t = dht.readTemperature()` | Reads temperature in Celsius using the DHT sensor alias supported by MbedO. |
| `map((int)t, 20, 40, 0, 255)` | Linearly scales temperature from the 20–40 °C range to the 0–255 PWM range. |
| `analogWrite(ENA_PIN, fanSpeed)` | Sends a PWM signal to the L298N ENA pin, controlling motor speed proportionally. |
| `fanPercent = map(fanSpeed, 0, 255, 0, 100)` | Converts the raw PWM value back to a human-readable percentage for the LCD. |
| Trailing `"  "` spaces | Overwrites stale characters when a shorter string replaces a longer one on the LCD. |

## Hardware & Safety Concept: PWM Motor Speed Control
PWM (Pulse Width Modulation) rapidly switches the motor voltage ON and OFF. The ratio of ON time to total period is the **duty cycle**. At 50% duty cycle the motor receives an average of half the supply voltage and spins at roughly half speed.
- The L298N H-bridge driver amplifies the Arduino's 5 V PWM signal to drive motors at higher voltages (up to 46 V, 2 A per channel).
- The two direction pins (IN1/IN2) set the H-bridge polarity; swapping their states reverses the motor.
- Never drive a motor directly from an Arduino pin — the current draw will exceed the pin's 40 mA limit and damage the microcontroller.

## Try This! (Challenges)
1. **Cooling Threshold**: Only run the fan if temperature exceeds 25 °C; below that, force `fanSpeed = 0` and print `"Fan: OFF"` on the LCD.
2. **Humidity Boost**: Read humidity with `dht.readHumidity()` and add a bonus to `fanSpeed` when humidity exceeds 70%, simulating a perceived-temperature (heat index) response.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Motor does not spin at all | ENA not connected | Verify D9 is wired to ENA; some L298N modules have ENA shorted by a jumper that must be removed. |
| LCD stays blank | Wrong I2C address | Try changing `0x27` to `0x3F` in the constructor. |
| Fan always at 0% | Temperature below 20 °C | Raise the DHT22 slider above 20 °C or lower the `map()` lower bound. |
| DHT22 returns NAN | Startup timing | Add `delay(500)` after `dht.begin()` to let the sensor initialise. |

## Mode Notes
All patterns used here (`dht.readTemperature()`, `map()`, `analogWrite`, `lcd.print`) are fully supported by MbedO interpreted mode.

## Related Projects
- [47 - Temp Humidity Serial](../intermediate/47-temp-humidity-serial.md)
- [86 - Weather Station](86-weather-station.md)
- [99 - Auto Night Light](99-auto-night-light.md)
- [101 - Automatic Gate](101-automatic-gate.md)
