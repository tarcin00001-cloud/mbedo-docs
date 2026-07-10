# 48 - Temp Alarm

Sound an acoustic alert on a buzzer when environmental temperatures exceed a safety limit.

## Goal
Learn how to read temperature data from a DHT22 sensor and trigger an emergency wailing alarm when a threshold setpoint is crossed.

## What You Will Build
When the temperature reading from the DHT22 sensor rises above 30C, the buzzer connected to pin D8 sounds a wailing siren. When it cools down, the buzzer is silenced.

**Why D2 and D8?** Pin D2 reads the DHT22 digital data packets. Pin D8 drives the wailing alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
#include <DHT.h>

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;
const int BUZZ_PIN = 8;

// Safety threshold in Celsius
const float TEMP_THRESHOLD = 30.0;

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("High Temperature Alarm System Ready");
  
  dht.begin();
}

void loop() {
  float tempC = dht.readTemperature();
  
  // Verify reading is valid before testing threshold
  if (isnan(tempC)) {
    Serial.println("Error: Sensor offline!");
  } else {
    Serial.print("Temperature: ");
    Serial.print(tempC);
    Serial.println("C");
    
    // Check if temperature exceeds threshold
    if (tempC > TEMP_THRESHOLD) {
      Serial.println("[WARNING] Temperature High! Sounding Siren!");
      
      // Sound alarm wail
      tone(BUZZ_PIN, 900);
      delay(150);
      tone(BUZZ_PIN, 1800);
      delay(150);
    } else {
      noTone(BUZZ_PIN);
      delay(1200); // Maintain 1.5s total loop rate
    }
  }
  
  delay(300); // Small loop cycle delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **Buzzer** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the DHT22, adjust the temperature slider above 30, and listen to the alarm siren.

## Expected Output

Terminal:
```
High Temperature Alarm System Ready
Temperature: 24.50C
Temperature: 32.50C
[WARNING] Temperature High! Sounding Siren!
...
```

### Expected Canvas Behavior

| Temperature Input | Reading vs Limit | Buzzer D8 State | Audio Character |
| --- | --- | --- | --- |
| Safe (< 30C) | Below | LOW - Silent | None |
| Hot (> 30C) | Above | Oscillating (900/1800 Hz) | Fast Siren |

The buzzer oscillates and vibrates on the canvas only when the temperature is hot.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `float tempC = dht.readTemperature()` | Reads the temperature value as a floating point number. |
| `if (tempC > TEMP_THRESHOLD)` | Evaluates if the current temperature exceeds our safety threshold. |

## Hardware & Safety Concept: Industrial Thermal Protection
Thermal alarms are critical fail-safes in battery storage rooms, server racks, and manufacturing environments. Standard procedures include generating auditory sirens, triggering ventilation fans, and shutting down power to machinery via isolation relays when safe operating boundaries are violated.

## Try This! (Challenges)
1. **Freeze Alarm**: Modify the code to sound a different alarm beep if the temperature drops below `5`C (representing a frost alarm).
2. **Hysteresis Protection**: Implement a hysteresis guard so that once the alarm starts, it doesn't turn off until the temperature drops below 28C.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Siren sounds constantly | Threshold too low or offline sensor | Ensure the sensor is reporting valid values in the Terminal. A failed read (`NAN`) will not trigger the alarm, but verify connections. |
| Clicking instead of tone | Ground wire loose | Verify Buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (DHT library calls and float thresholds driving wailing alarms) are supported by MbedO interpreted mode.

## Related Projects
- [41 - Thermostat Relay](41-thermostat-relay.md)
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [50 - High Temp Fan](50-high-temp-fan.md)
