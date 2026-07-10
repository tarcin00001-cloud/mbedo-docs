# 121 - Greenhouse Controller

Build an automated greenhouse monitoring system that controls a ventilation fan (via a relay) based on DHT22 temperature thresholds, and tracks soil moisture (via a water level sensor) to sound alerts when irrigation is required.

## Goal
Learn how to manage multiple environmental thresholds (temperature and moisture level) simultaneously, displaying structured logs on a 16x2 LCD while operating relays and sirens in interpreted mode.

## What You Will Build
A climate and irrigation console:
- **Temperature Ventilation**: If the temperature climbs above 30°C, a relay turns on a cooling fan.
- **Moisture Alert**: If the water/moisture sensor value falls below 300 (out of 1023), a warning buzzer alerts the operator.
- **LCD Readout**: Displays current temperature, humidity, and water level, alongside status codes (e.g. `FAN:ON` or `WATER ALERT`).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| Water Level Sensor | `water_level` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 16x2 I2C LCD | `lcd_i2c` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply |
| DHT22 Sensor | SDA | D2 | Digital data |
| DHT22 Sensor | GND | GND | Ground reference |
| Water Level Sensor | VCC | 5V | Power supply |
| Water Level Sensor | SIG | A1 | Analog signal |
| Water Level Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply |
| Relay Module | IN | D8 | Relay control (HIGH = Fan ON) |
| Relay Module | GND | GND | Ground reference |
| Active Buzzer | VCC | D7 | Buzzer drive pin |
| Active Buzzer | GND | GND | Ground reference |
| 16x2 I2C LCD | VCC | 5V | Power supply |
| 16x2 I2C LCD | SDA | A4 | I2C Data bus |
| 16x2 I2C LCD | SCL | A5 | I2C Clock bus |
| 16x2 I2C LCD | GND | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <LiquidCrystal_I2C.h>
#include <DHT.h>

const int DHT_PIN    = 2;
const int DHT_TYPE   = DHT22;
const int WATER_PIN  = A1;
const int RELAY_PIN  = 8;
const int BUZZER_PIN = 7;

const float TEMP_LIMIT    = 30.0;
const int WATER_LIMIT     = 300;

DHT dht(DHT_PIN, DHT_TYPE);
LiquidCrystal_I2C lcd(0x27, 16, 2);

void setup() {
  Serial.begin(9600);
  dht.begin();
  
  pinMode(RELAY_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  digitalWrite(RELAY_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);

  lcd.init();
  lcd.backlight();
  lcd.print("Greenhouse Init");
  delay(1500);
  lcd.clear();

  Serial.println("Greenhouse Controller Active");
}

void loop() {
  float humidity = dht.readHumidity();
  float tempC = dht.readTemperature();
  int waterValue = analogRead(WATER_PIN);

  if (isnan(humidity) || isnan(tempC)) {
    Serial.println("Error: DHT22 read failure!");
    return;
  }

  // Print values to Serial
  Serial.print("Temp: ");    Serial.print(tempC);     Serial.print("C | ");
  Serial.print("Hum: ");     Serial.print(humidity);  Serial.print("% | ");
  Serial.print("Moist: ");   Serial.println(waterValue);

  // 1. Ventilation control
  if (tempC > TEMP_LIMIT) {
    digitalWrite(RELAY_PIN, HIGH); // Turn fan on
    Serial.println(" -> Ventilation Fan ON");
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Turn fan off
  }

  // 2. Moisture alert control
  if (waterValue < WATER_LIMIT) {
    digitalWrite(BUZZER_PIN, HIGH);
    Serial.println(" -> warning: Low Moisture alert!");
  } else {
    digitalWrite(BUZZER_PIN, LOW);
  }

  // 3. LCD screen output
  // Line 1: Temp & Humidity
  lcd.setCursor(0, 0);
  lcd.print("T:");
  lcd.print(tempC, 0);
  lcd.print("C H:");
  lcd.print(humidity, 0);
  lcd.print("% ");
  if (tempC > TEMP_LIMIT) {
    lcd.print("FAN:ON ");
  } else {
    lcd.print("FAN:OFF");
  }

  // Line 2: Moisture status
  lcd.setCursor(0, 1);
  lcd.print("Moist:");
  lcd.print(waterValue);
  lcd.print(" ");
  if (waterValue < WATER_LIMIT) {
    lcd.print("WANT WATER");
  } else {
    lcd.print("WATER:OK  ");
  }

  delay(1000);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, **Water Level Sensor**, **Relay Module**, **Active Buzzer**, and **16x2 I2C LCD** onto the canvas.
2. Connect DHT22: **VCC** to **5V**, **SDA** to **D2**, **GND** to **GND**.
3. Connect Water Level: **VCC** to **5V**, **SIG** to **A1**, **GND** to **GND**.
4. Connect Relay: **VCC** to **5V**, **IN** to **D8**, **GND** to **GND**.
5. Connect Buzzer: **VCC** to **D7**, **GND** to **GND**.
6. Connect LCD: **VCC** to **5V**, **SDA** to **A4**, **SCL** to **A5**, **GND** to **GND**.
7. Paste code, select the interpreted mode, and click **Run**.
8. Modify DHT22 temperature above 30°C to activate the relay, and slide the water level sensor below 300 to activate the buzzer.

## Expected Output

Terminal:
```
Greenhouse Controller Active
Temp: 28.50C | Hum: 62.00% | Moist: 500
Temp: 32.00C | Hum: 58.00% | Moist: 250
 -> Ventilation Fan ON
 -> warning: Low Moisture alert!
```

LCD Display:
```
T:32C H:58% FAN:ON 
Moist:250 WANT WATER
```

## Expected Canvas Behavior
| DHT22 Temp | Water Level (A1) | Relay (D8) | Buzzer (D7) | LCD Line 1 | LCD Line 2 |
| --- | --- | --- | --- | --- | --- |
| 28.0 C | 450 | LOW | LOW | `T:28C H:60% FAN:OFF` | `Moist:450 WATER:OK` |
| 33.0 C | 450 | HIGH | LOW | `T:33C H:60% FAN:ON` | `Moist:450 WATER:OK` |
| 28.0 C | 180 | LOW | HIGH | `T:28C H:60% FAN:OFF` | `Moist:180 WANT WATER` |
| 33.0 C | 180 | HIGH | HIGH | `T:33C H:60% FAN:ON` | `Moist:180 WANT WATER` |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tempC > TEMP_LIMIT` | Checks if current temp value exceeds safe bounds, changing state of ventilation control. |
| `waterValue < WATER_LIMIT` | Verifies if moisture level drops below the safety margin, triggering buzzer warning. |
| `lcd.print("FAN:ON ")` | Prints clear, real-time diagnostic flags to the operator console. |

## Hardware & Safety Concept: Automated Climate Controls
Greenhouse telemetry requires simultaneous air and soil metrics to stabilize plant growth. Relying on digital controllers like the Arduino allows us to deploy automatic feedback loops where relays operate ventilators and buzzers coordinate manual intervention when automatic compensation fails.

## Try This! (Challenges)
1. **Dynamic Fan Speed**: Replace the simple on/off relay control with a transistor drive circuit on pin D6 (using `analogWrite()`) to run the fan speed proportional to the temperature excess.
2. **Humidifier Control**: Connect a second relay to pin D9 and program it to activate a humidifier if the relative humidity drops below 40%.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LCD output is corrupt or shifting | Too many prints | Ensure that `lcd.setCursor` is run before every print and trailing spaces clear old digits. |
| Buzzer sounds constantly | Weak connection or floating input | Check that the water level ground pin is correctly tied to the common ground rail. |

## Mode Notes
This multi-sensor control logic runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [40 - Water Pump Control](../intermediate/40-water-pump-control.md)
- [50 - High Temp Fan](../intermediate/50-high-temp-fan.md)
- [90 - Smart Thermostat](../advanced/90-smart-thermostat.md)
