# 65 - DS18B20 Alarm

Sound an acoustic alert on a buzzer when a DS18B20 sensor measures temperatures above 35C.

## Goal
Learn how to read temperature data from a One-Wire sensor and trigger an emergency wailing alarm when a threshold setpoint is crossed.

## What You Will Build
When the temperature reading from the DS18B20 sensor rises above 35C, the buzzer connected to pin D8 sounds a wailing siren. When it cools down, the buzzer is silenced.

**Why D2 and D8?** Pin D2 reads the DS18B20 digital data packets. Pin D8 drives the wailing alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DS18B20 Temp Sensor | `ds18b20` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |
| 4.7k ohm resistor | `resistor` | Optional | Yes (pull-up for DQ) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DS18B20 Sensor | VCC | 5V | Power supply (5V) |
| DS18B20 Sensor | DQ | D2 | Data connection |
| DS18B20 Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
#include <OneWire.h>
#include <DallasTemperature.h>

const int ONE_WIRE_BUS = 2;
const int BUZZ_PIN     = 8;

// Safety threshold in Celsius
const float TEMP_THRESHOLD = 35.0;

OneWire oneWire(ONE_WIRE_BUS);
DallasTemperature sensors(&oneWire);

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("DS18B20 Temperature Alert System Ready");
  
  sensors.begin();
}

void loop() {
  sensors.requestTemperatures();
  float tempC = sensors.getTempCByIndex(0);
  
  if (tempC == DEVICE_DISCONNECTED_C) {
    Serial.println("Error: Sensor offline!");
  } else {
    Serial.print("Temperature: ");
    Serial.print(tempC);
    Serial.println(" C");
    
    // Check if temperature exceeds threshold
    if (tempC > TEMP_THRESHOLD) {
      Serial.println("[WARNING] Temperature High! Sounding Siren!");
      
      // Sound alarm wail
      tone(BUZZ_PIN, 1000);
      delay(150);
      tone(BUZZ_PIN, 1500);
      delay(150);
    } else {
      noTone(BUZZ_PIN);
      delay(700); // Maintain 1-second refresh rate
    }
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DS18B20 Sensor**, and **Buzzer** onto the canvas.
2. Connect DS18B20 **VCC** to Arduino **5V**, **DQ** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the DS18B20, adjust the temperature slider above 35, and listen to the alarm siren.

## Expected Output

Terminal:
```
DS18B20 Temperature Alert System Ready
Temperature: 24.50 C
Temperature: 36.80 C
[WARNING] Temperature High! Sounding Siren!
...
```

### Expected Canvas Behavior

| Temperature Input | Reading vs Limit | Buzzer D8 State | Audio Character |
| --- | --- | --- | --- |
| Safe (< 35C) | Below | LOW - Silent | None |
| Hot (> 35C) | Above | Oscillating (1000/1500 Hz) | Fast Siren |

The buzzer vibrates on the canvas only when the temperature is hot.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `float tempC = sensors.getTempCByIndex(0)` | Reads the temperature value as a floating point number. |
| `if (tempC > TEMP_THRESHOLD)` | Evaluates if the current temperature exceeds our safety threshold. |

## Hardware & Safety Concept: Industrial Thermal Protection
The DS18B20 is a highly accurate, factory-calibrated digital sensor (accuracy within +/-0.5C). It is encapsulated in waterproof stainless steel probes, making it the industry standard for monitoring liquids, soil, and pipe networks in breweries, food manufacturing, and chemical storage.

## Try This! (Challenges)
1. **Freeze Alarm**: Modify the code to sound a different alarm beep if the temperature drops below `5`C (representing a frost alarm).
2. **Hysteresis Protection**: Implement a hysteresis guard so that once the alarm starts, it doesn't turn off until the temperature drops below 33C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Siren sounds constantly | Threshold too low or offline sensor | Ensure the sensor is reporting valid values in the Terminal. A failed read (`-127`) will be filtered by the `DEVICE_DISCONNECTED_C` check. |
| Clicking instead of tone | Ground wire loose | Verify Buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (DS18B20 sensor reads and float thresholds driving wailing alarms) are supported by MbedO interpreted mode.

## Related Projects
- [48 - Temp Alarm](48-temp-alarm.md)
- [64 - DS18B20 Temp Print](64-ds18b20-temp-print.md)
- [66 - DS18B20 LCD](66-ds18b20-lcd.md)
