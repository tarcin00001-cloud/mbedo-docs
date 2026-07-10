# 49 - Humidity Alert LED

Turn on a warning LED when relative humidity levels exceed a comfortable range.

## Goal
Learn how to read humidity levels from a DHT22 sensor and trigger a digital indicator LED when relative humidity exceeds a setpoint limit.

## What You Will Build
When the relative humidity measured by the DHT22 exceeds 60%, the LED connected to D13 turns ON, indicating high humidity (representing damp/mold risk or greenhouse limits). When humidity is safe, the LED turns OFF.

**Why D2 and D13?** Pin D2 reads the digital DHT22 sensor data. Pin D13 controls the warning indicator LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| DHT22 Sensor | `dht22` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |
| 10k ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| DHT22 Sensor | VCC | 5V | Power supply (5V) |
| DHT22 Sensor | SDA | D2 | Data connection |
| DHT22 Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
#include <DHT.h>

const int DHT_PIN  = 2;
const int DHT_TYPE = DHT22;
const int LED_PIN  = 13;

// Comfort threshold for relative humidity (%)
const float HUMIDITY_THRESHOLD = 60.0;

DHT dht(DHT_PIN, DHT_TYPE);

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Humidity Alert System Ready");
  
  dht.begin();
}

void loop() {
  float humidity = dht.readHumidity();
  
  if (isnan(humidity)) {
    Serial.println("Error: Failed to read humidity!");
  } else {
    Serial.print("Humidity: ");
    Serial.print(humidity);
    Serial.println("%");
    
    // Check if humidity exceeds threshold
    if (humidity > HUMIDITY_THRESHOLD) {
      digitalWrite(LED_PIN, HIGH);
      Serial.println("[WARNING] High Humidity! LED ON");
    } else {
      digitalWrite(LED_PIN, LOW);
    }
  }
  
  delay(1500); // 1.5 second poll delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **DHT22 Sensor**, and **LED** onto the canvas.
2. Connect DHT22 **VCC** to Arduino **5V**, **SDA** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect LED **A** to Arduino **D13** and LED **C** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the DHT22 sensor, adjust the humidity slider above 60%, and watch the LED turn on.

## Expected Output

Terminal:
```
Humidity Alert System Ready
Humidity: 45.20%
Humidity: 65.80%
[WARNING] High Humidity! LED ON
...
```

### Expected Canvas Behavior

| Humidity Slider | Value vs Limit | Pin D13 Output | LED State |
| --- | --- | --- | --- |
| Low (< 60%) | Below | LOW (0V) | OFF |
| High (> 60%) | Above | HIGH (5V) | ON |

The LED on the canvas lights up immediately when the humidity slider crosses 60%.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `float humidity = dht.readHumidity()` | Reads relative humidity value as a float. |
| `if (humidity > HUMIDITY_THRESHOLD)` | Evaluates if relative humidity exceeds the comfort threshold. |

## Hardware & Safety Concept: Humidity and Condensation
Relative humidity represents the ratio of water vapor present in the air to the maximum amount the air could hold at that temperature. In high-humidity environments (above 60-70%), mold spores germinate rapidly, and condensation can form on cold metal surfaces, threatening electronic components with corrosion and short-circuits.

## Try This! (Challenges)
1. **Flash Alert**: Make the LED flash on and off repeatedly while the humidity is high, to draw more attention.
2. **Dehumidifier indicator**: Reverse the logic so that the LED is normally ON (representing safe conditions) and turns OFF only when high humidity is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Threshold is set too low | Adjust the `HUMIDITY_THRESHOLD` constant to 70% or 80%. |
| LED never turns on | LED wired backwards | Check anode (A) connects to D13 and cathode (C) connects to GND. |

## Mode Notes
These patterns (reading float humidity levels driving conditional digital outputs) are supported by MbedO interpreted mode.

## Related Projects
- [47 - Temp Humidity Serial](47-temp-humidity-serial.md)
- [50 - High Temp Fan](50-high-temp-fan.md)
