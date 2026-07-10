# 87 - Air Quality Monitor

Combine an MQ-2 gas/smoke sensor and a DHT22 temperature/humidity sensor to display a basic air quality report on an I2C LCD, with a buzzer alert on hazardous readings.

## Goal
Learn how to combine analog gas sensor readings and digital temperature/humidity data into a unified safety monitoring system with a visual display and audio alert.

## What You Will Build
The LCD displays current temperature, humidity, and gas concentration level. If the gas sensor reading rises above a hazard threshold, the buzzer sounds an alert tone, the display shows "ALERT!" on line 2, and the system holds the alarm for 3 seconds.

**Why A0, D2, and D8?** Pin A0 reads the analog MQ-2 voltage. Pin D2 communicates with the DHT22. Pin D8 drives the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MQ-2 Gas Sensor | `mq2_sensor` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes (DHT pull-up) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Power supply |
| MQ-2 Sensor | A0 | A0 | Analog signal output |
| MQ-2 Sensor | GND | GND | Ground reference |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Single-wire data |
| DHT22 Sensor | GND | GND | Ground reference |
| Buzzer | + | D8 | Signal pin |
| Buzzer | - | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int GAS_PIN    = A0;
const int DHT_PIN    = 2;
const int BUZZER_PIN = 8;
const int DHT_TYPE   = DHT22;

const int GAS_THRESHOLD = 500; // ADC units (0-1023); tune for your environment

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  dht.begin();
  pinMode(BUZZER_PIN, OUTPUT);
  
  lcd.init();
  lcd.backlight();
  lcd.print("Air Monitor");
  delay(1500);
  lcd.clear();
  
  Serial.println("Air Quality Monitor Active");
}

void loop() {
  int gasLevel  = analogRead(GAS_PIN);
  float humidity = dht.readHumidity();
  float tempC   = dht.readTemperature();
  
  Serial.print("Gas: ");    Serial.print(gasLevel);
  Serial.print(" | Temp: "); Serial.print(tempC);
  Serial.print(" C | Hum: "); Serial.print(humidity); Serial.println("%");
  
  // Line 1: Temperature and humidity
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(tempC, 0);
  lcd.print("C H:");
  lcd.print(humidity, 0);
  lcd.print("%   ");
  
  if (gasLevel > GAS_THRESHOLD) {
    // Gas hazard alert
    tone(BUZZER_PIN, 880, 500);
    lcd.setCursor(0, 1);
    lcd.print("!! GAS ALERT !!!");
    delay(3000);
    noTone(BUZZER_PIN);
  } else {
    // Normal operation
    noTone(BUZZER_PIN);
    lcd.setCursor(0, 1);
    lcd.print("Gas:");
    lcd.print(gasLevel);
    lcd.print(" OK     ");
  }
  
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MQ-2 Sensor**, **DHT22 Sensor**, **16x2 I2C LCD**, and **Buzzer** onto the canvas.
2. Connect MQ-2: **VCC** to **5V**, **A0** to **A0**, **GND** to **GND**.
3. Connect DHT22: **VCC** to **5V**, **SDA** to **D2**, **GND** to **GND**.
4. Connect Buzzer: **+** to **D8**, **-** to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor.
7. Use the default Arduino interpreted runtime.
8. Click **Run**.
9. Adjust the MQ-2 slider above 500. Watch the LCD show the alert and the buzzer sound.

## Expected Output

Terminal:
```
Air Quality Monitor Active
Gas: 120 | Temp: 24 C | Hum: 60%
Gas: 650 | Temp: 24 C | Hum: 60%
```

LCD (Normal):
```
T:24C H:60%
Gas:120 OK
```

LCD (Alert):
```
T:24C H:60%
!! GAS ALERT !!!
```

## Expected Canvas Behavior

| MQ-2 Slider Value | Comparison | Buzzer State | LCD Line 2 |
| --- | --- | --- | --- |
| < 500 | Below threshold | Silent | `Gas:XXX OK` |
| > 500 | Above threshold | 880 Hz tone | `!! GAS ALERT !!!` |

## Code Walkthrough

| Expression | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, 880, 500)` | Plays an 880 Hz tone for 500 ms on the buzzer. |
| `GAS_THRESHOLD = 500` | Tunable constant. Raise for less sensitive environments. Lower for more sensitive environments. |

## Hardware & Safety Concept: Calibrating Gas Sensors
The MQ-2 sensor output voltage drifts based on environmental conditions (temperature, humidity, and baseline air quality).
- In a real deployment, you must place the sensor in **clean air** for 48 hours to establish a baseline reading.
- Readings above the baseline by a fixed factor (usually 2x to 5x) are then used as alert thresholds.
- In simulation, you set thresholds empirically by observing the ADC values under normal vs. simulated hazard conditions.

## Try This! (Challenges)
1. **LED Traffic Light**: Wire three LEDs (green, yellow, red) to D3, D4, D5. Light green for gas < 300, yellow for 300-500, and red for > 500.
2. **Log to Terminal**: Prefix each Terminal print with a timestamp counter (using `millis()`) to create a time-stamped event log.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alert never triggers | Threshold set too high | Lower `GAS_THRESHOLD` to 300 or watch the Terminal to see actual raw values. |
| Gas reading stuck at 0 | MQ-2 not connected to A0 | Verify the MQ-2 analog output pin connects to Arduino A0. |

## Mode Notes
These patterns (analog reads, threshold comparisons, `tone()`, and LCD output) are supported by MbedO interpreted mode.

## Related Projects
- [86 - Weather Station](86-weather-station.md)
- [63 - Pressure Alarm](../intermediate/63-pressure-alarm.md)
- [88 - Fire Safety System](88-fire-safety-system.md)
