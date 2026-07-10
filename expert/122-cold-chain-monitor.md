# 122 - Cold Chain Monitor

Build a precision temperature logging and alert terminal for cold storage logistics using a DS18B20 one-wire temperature sensor, an active buzzer, a control relay, and an I2C LCD screen.

## Goal
Learn how to read temperatures from a digital one-wire bus using the DallasTemperature library, implement a high-precision threshold alarm, and actuate safety switches (relay and buzzer) for thermal runaway protection.

## What You Will Build
A medical/food cold chain logger:
- **DS18B20 Temp Probe**: Reads temperature on a single digital pin.
- **Relay Cut-off**: If the temperature climbs above -10.0°C (standard food/vaccine storage limit), a relay switches on (simulating a backup cooling loop or alarm signal).
- **Buzzer Alarm**: Emits a warning pulse if the temperature breaches safety levels.
- **LCD Status Display**: Prints current temperature with decimal precision and displays `SAFE` or `ALARM!`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DS18B20 Sensor | `ds18b20` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |
| 4.7k ohm resistor | `resistor` | Optional | Yes (pull-up resistor) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 Sensor | VCC | 5V | Power supply |
| DS18B20 Sensor | DQ | D2 | One-wire digital bus (needs 4.7k pull-up to VCC) |
| DS18B20 Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D8 | Relay control |
| Relay Module | GND | GND | Ground reference |
| Active Buzzer | VCC | D7 | Buzzer drive pin |
| Active Buzzer | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>
#include <Wire.h>
#include <LiquidCrystal_I2C.h>

const int ONE_WIRE_BUS = 2;
const int RELAY_PIN    = 8;
const int BUZZER_PIN   = 7;

const float TEMP_LIMIT = -10.0; // Critical temperature threshold

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  sensors.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Cold Chain Init");
  delay(1500);
  lcd.clear();

  Serial.println("Cold Chain Monitor Online");
}

void loop() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: DS18B20 Disconnected!");
    lcd.setCursor(0, 0);
    lcd.print("SENSOR ERROR!   ");
    return;
  }

  Serial.print("Temp: ");
  Serial.print(tempC);
  Serial.println(" C");

  // Threshold logic
  if (tempC > TEMP_LIMIT) {
    digitalWrite(RELAY_PIN, HIGH);  // Activate emergency cooling loop
    digitalWrite(BUZZER_PIN, HIGH); // Alarm siren
    
    lcd.setCursor(0, 0);
    lcd.print("TEMP CRITICAL!  ");
    lcd.setCursor(0, 1);
    lcd.print("T: ");
    lcd.print(tempC, 1);
    lcd.print("C  ALARM    ");
  } else {
    digitalWrite(RELAY_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);

    lcd.setCursor(0, 0);
    lcd.print("Temp Secure     ");
    lcd.setCursor(0, 1);
    lcd.print("T: ");
    lcd.print(tempC, 1);
    lcd.print("C  SAFE     ");
  }

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DS18B20 Sensor**, **Relay Module**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect DS18B20: **VCC** to **5V**, **DQ** to **D2**, **GND** to **GND**.
3. Connect Relay: **VCC** to **5V**, **IN** to **D8**, **GND** to **GND**.
4. Connect Buzzer: **VCC** to **D7**, **GND** to **GND**.
5. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
6. Paste the code into the editor, select interpreted mode, and click **Run**.
7. Change the slider on the DS18B20 sensor. Shift it above -10°C to watch the alarms trip.

## Expected Output

Terminal:
```
Cold Chain Monitor Online
Temp: -18.50 C
Temp: -8.20 C
```

LCD Display:
```
TEMP CRITICAL!  
T: -8.2C  ALARM    
```

## Expected Canvas Behavior
| DS18B20 Temperature | Relay (D8) | Buzzer (D7) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- |
| -20.0 C | LOW | LOW | `Temp Secure` | `T: -20.0C  SAFE` |
| -15.5 C | LOW | LOW | `Temp Secure` | `T: -15.5C  SAFE` |
| -9.5 C | HIGH | HIGH | `TEMP CRITICAL!` | `T: -9.5C  ALARM` |
| 5.0 C | HIGH | HIGH | `TEMP CRITICAL!` | `T: 5.0C  ALARM` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `sensors.requestTemperatures()` | Sends a broadcast signal over the one-wire bus telling all probes to convert and save temperatures. |
| `sensors.getTempCByIndex(0)` | Reads the raw binary scratchpad value from the first sensor resolved on the bus and returns Celsius. |
| `tempC > TEMP_LIMIT` | Evaluates if the environment is warmer than the safety threshold (since -9.0 is greater than -10.0). |

## Hardware & Safety Concept: One-Wire Bus Pull-Up
The Dallas DS18B20 sensor communicates over a proprietary 1-Wire protocol. Because the single data line is open-drain, it requires a 4.7k ohm physical pull-up resistor to VCC (5V) to pull the signal line HIGH when the master or slave devices are not actively pulling it LOW. Without this pull-up resistor, communication fails completely and returns `DEVICE_DISCONNECTED_C`.

## Try This! (Challenges)
1. **Critical Hysteresis**: Implement a 1.0°C hysteresis window so that once the temperature breaches -10.0°C and triggers the alarm, it must cool back down below -11.0°C before the relay and buzzer turn off. This stops the relay from clicking rapidly near the limit.
2. **Periodic Warning Tone**: Modify the alarm phase so that the buzzer emits a brief 1-second pulse every 5 seconds rather than sounding continuously.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Reads `DEVICE_DISCONNECTED_C` | Missing pull-up resistor or wrong wiring | Connect data pin D2 to a 4.7k pull-up resistor or ensure the MbedO schematic aligns. |
| LCD does not print temp | LCD initialisation error | Verify that the I2C wires (SDA/SCL) are connected to A4/A5. |

## Mode Notes
The DallasTemperature shims are supported in MbedO interpreted mode.

## Related Projects
- [64 - DS18B20 Temp Print](../intermediate/64-ds18b20-temp-print.md)
- [65 - DS18B20 Alarm](../intermediate/65-ds18b20-alarm.md)
- [66 - DS18B20 LCD](../intermediate/66-ds18b20-lcd.md)
