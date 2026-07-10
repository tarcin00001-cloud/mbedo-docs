# 82 - Pico DS18B20 Alarm

Sound a waterproof probe warning siren if liquid temperature levels exceed a limit.

## Goal
Learn how to use digital probe sensors with One-Wire logic to trigger acoustic indicators.

## What You Will Build
A liquid over-temperature alarm system:
- **DS18B20 Sensor (GP12)**: Monitors liquid temperature.
- **Active Buzzer (GP14)**: Sounds a warning pulse if the temperature exceeds 40.0°C.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DS18B20 Temp Sensor | `ds18b20` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 4.7k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 | VCC | 3.3V | Power supply |
| DS18B20 | DQ | GP12 | Data signal (requires 4.7k pull-up to 3.3V) |
| DS18B20 | GND | GND | Ground reference |
| Active Buzzer | VCC (+) | GP14 | Alarm pin |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONE_WIRE_BUS = 12;
const int BUZZER_PIN   = 14;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

const float ALARM_LIMIT = 40.0;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  sensors.begin();
}

void loop() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);

  if (tempC != DEVICE_DISCONNECTED_C) {
    if (tempC > ALARM_LIMIT) {
      // Rapid warning beeps
      digitalWrite(BUZZER_PIN, HIGH);
      delay(150);
      digitalWrite(BUZZER_PIN, LOW);
      delay(150);
    } else {
      digitalWrite(BUZZER_PIN, LOW);
    }
  }

  delay(700);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DS18B20 Sensor**, and **Active Buzzer** onto the canvas.
2. Connect DS18B20 to **GP12** (with 4.7k pull-up to 3V3), Buzzer positive to **GP14**, GND lines to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature slider on the DS18B20 model above 40°C to activate the alarm.

## Expected Output

Terminal:
```
Simulation active. Monitoring DS18B20 on GP12. Alarm: 40C.
```

## Expected Canvas Behavior
| DS18B20 Temperature | GP14 (Buzzer) State | Status |
| --- | --- | --- |
| < 40.0 C | LOW | Safe |
| > 40.0 C | Pulsing HIGH/LOW | **OVER-TEMP ALARM ACTIVE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tempC > ALARM_LIMIT` | Checks if the temperature read from probe 0 exceeds the 40°C safety limit. |

## Hardware & Safety Concept: Sensor Waterproofing
The DS18B20 sensor is available in a stainless steel tube probe shell. The sensor chip is sealed inside the tube with thermal conductive epoxy, making it waterproof and suitable for measuring temperatures in soil, water, or corrosive liquids. In contrast, sensors like the DHT22 or BMP180 are open to the air and will short-circuit immediately if exposed to moisture.

## Try This! (Challenges)
1. **Freeze Siren**: Modify the code to sound the alarm if the temperature drops below 0.0°C.
2. **Alert indicator**: Add a Red LED on GP15 that flashes in sync with the buzzer alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm beeps constantly on boot | Read error | When the sensor fails to start, it returns `-127.00`. Confirm you check `tempC != DEVICE_DISCONNECTED_C` before evaluating thresholds. |

## Mode Notes
This basic threshold logic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [81 - Pico DS18B20 Serial](81-pico-ds18b20-serial.md)
- [83 - Pico DS18B20 LCD](83-pico-ds18b20-lcd.md)
