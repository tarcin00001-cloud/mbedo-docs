# 72 - Pico DHT Alarm

Sound an acoustic alert if ambient temperature levels exceed a safety threshold.

## Goal
Learn how to use digital sensor telemetry to trigger safety actuator loops (buzzers) based on threshold logic.

## What You Will Build
An over-temperature warning siren:
- **DHT22 Sensor (GP12)**: Monitors room temperature.
- **Active Buzzer (GP14)**: Sounds a pulsing siren (200 ms ON, 200 ms OFF) if the temperature rises above 32.0°C.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| DHT22 Sensor Module | `dht22` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 10k-ohm Resistor | `resistor` | Optional | Yes (pull-up) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| DHT22 | VCC | 3.3V | Power supply |
| DHT22 | SDA | GP12 | Data signal (requires 10k pull-up to VCC) |
| DHT22 | GND | GND | Ground reference |
| Active Buzzer | VCC (+) | GP14 | Alarm pin |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
#include <DHT.h>

const int DHT_PIN    = 12;
const int BUZZER_PIN = 14;
#define DHTTYPE DHT22

DHT dht(DHT_PIN, DHTTYPE);

const float TEMP_LIMIT = 32.0;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  dht.begin();
}

void loop() {
  float temp = dht.readTemperature();

  // Validate reading
  if (!isnan(temp)) {
    if (temp > TEMP_LIMIT) {
      // Sound Alarm (2 beeps per loop)
      digitalWrite(BUZZER_PIN, HIGH);
      delay(200);
      digitalWrite(BUZZER_PIN, LOW);
      delay(200);
    } else {
      digitalWrite(BUZZER_PIN, LOW); // Safe
    }
  }

  delay(1600); // Remaining delay to fit DHT22 2-second timing window
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **DHT22 Sensor**, and **Active Buzzer** onto the canvas.
2. Connect DHT22 to **GP12** (with 10k pull-up to 3V3), Buzzer positive to **GP14**, GND lines to **GND**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Slide the temperature slider on the DHT22 model to 35°C to sound the alarm.

## Expected Output

Terminal:
```
Simulation active. Monitoring temperature on GP12. Threshold: 32C.
```

## Expected Canvas Behavior
| DHT22 Temperature | GP14 (Buzzer) State | Status |
| --- | --- | --- |
| < 32.0 C | LOW (OFF) | System Safe |
| > 32.0 C | Pulsing HIGH/LOW | **OVER-TEMP ALARM ACTIVE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `temp > TEMP_LIMIT` | Evaluates if the Celsius temperature exceeds the safety threshold of 32.0°C. |

## Hardware & Safety Concept: Sensor Thermal Lag
Air temperature does not change instantly. Because of this **thermal lag** (the delay for heat to pass through the plastic body of the sensor to the internal chip), temperature sensors can take 10 to 30 seconds to react to sudden air temperature changes. For critical systems, fast-acting thermistors with minimal casing are preferred over plastic modules like the DHT22.

## Try This! (Challenges)
1. **Visual Warning**: Add a Red LED on GP15 that flashes in sync with the buzzer alarm.
2. **Hysteresis Guard**: Modify the code to turn the alarm ON if temp > 32.0C, but only turn it OFF once the temperature drops below 30.0°C to prevent rapid switching.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds constantly even when cool | Threshold set too low | Check the value of `TEMP_LIMIT` in the code and verify it matches the ambient temperature. |

## Mode Notes
This basic threshold logic project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [71 - Pico DHT Serial](71-pico-dht-serial.md)
- [73 - Pico DHT LCD](73-pico-dht-lcd.md)
