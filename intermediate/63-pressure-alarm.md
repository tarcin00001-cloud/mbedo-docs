# 63 - Pressure Alarm

Sound a warning alarm on a buzzer when barometric pressure drops rapidly, indicating an incoming storm.

## Goal
Learn how to read I2C pressure data, monitor values for sudden drops, and trigger an auditory alarm warning using conditional comparison checks.

## What You Will Build
The system monitors barometric pressure. If the atmospheric pressure drops below 1000 hPa (indicating low pressure and unstable weather), the buzzer connected to pin D8 sounds a repeating alert tone. When pressure is stable, the buzzer is silent.

**Why A4, A5, and D8?** Pins A4/A5 read the BMP180 sensor via I2C. Pin D8 controls the storm alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| BMP180 Sensor | `bmp180` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| BMP180 Sensor | VCC | 3.3V | Power supply |
| BMP180 Sensor | SDA | A4 | I2C Serial Data connection |
| BMP180 Sensor | SCL | A5 | I2C Serial Clock connection |
| BMP180 Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
#include <Wire.h>
#include <Adafruit_BMP085.h>

const int BUZZ_PIN = 8;
Adafruit_BMP085 bmp;

// Threshold for storm warning (hectopascals)
const float STORM_LIMIT_HPA = 1000.0;

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Storm Alarm Monitor Ready");
  
  if (!bmp.begin()) {
    Serial.println("Error: BMP180 initialization failed!");
    while (1) {}
  }
}

void loop() {
  int32_t pressurePa = bmp.readPressure();
  float pressureHPa = pressurePa / 100.0;
  
  Serial.print("Pressure: ");
  Serial.print(pressureHPa);
  Serial.println(" hPa");
  
  // Trigger alarm if pressure drops below threshold
  if (pressureHPa < STORM_LIMIT_HPA) {
    Serial.println("[WARNING] Low Pressure! Storm Alert!");
    
    // Play warning tone
    tone(BUZZ_PIN, 800);
    delay(200);
    noTone(BUZZ_PIN);
    delay(200);
  } else {
    noTone(BUZZ_PIN);
    delay(1000); // Poll rate delay
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **BMP180 Sensor**, and **Buzzer** onto the canvas.
2. Connect BMP180 **VCC** to Arduino **3.3V**, **GND** to Arduino **GND**, **SDA** to Arduino **A4**, and **SCL** to Arduino **A5**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the BMP180 sensor, adjust the pressure slider below 1000, and watch the alarm beeper trigger.

## Expected Output

Terminal:
```
Storm Alarm Monitor Ready
Pressure: 1013.25 hPa
Pressure: 995.50 hPa
[WARNING] Low Pressure! Storm Alert!
...
```

### Expected Canvas Behavior

| Pressure Slider State | Value vs Limit | Buzzer State (D8) | Audio Character |
| --- | --- | --- | --- |
| Stable (> 1000 hPa) | Above | LOW - Silent | None |
| Low (< 1000 hPa) | Below | Pulsed (800 Hz) | Slow Alarm Beeps |

The buzzer vibrates and flashes on the canvas only when the pressure falls below the threshold.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `float pressureHPa = pressurePa / 100.0` | Converts raw Pascals to hectopascals (millibars). |
| `if (pressureHPa < STORM_LIMIT_HPA)` | Active-low warning check. A drop in air pressure below 1000 hPa indicates low pressure, triggering the alarm. |

## Hardware & Safety Concept: Barometric Pressure & Weather
Rapid changes in atmospheric pressure are prime indicators of weather shifts.
- A sudden **rise** in pressure indicates dry, stable, and clear weather.
- A sudden **drop** in pressure indicates wind, storm, or heavy rain.
Industrial weather stations log pressure gradients over hours to forecast weather trends and warn operators of high-winds or storms.

## Try This! (Challenges)
1. **Flash Alert**: Wire an LED to D13 and make it blink in synchronization with the beeper pulses.
2. **Hysteresis Guard**: Implement a 2 hPa hysteresis guard so that once the alarm triggers, it doesn't turn off until pressure rises back above 1002 hPa.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Beeper stays on constantly | Threshold is set too high | Check the raw pressure value printed in the Terminal, and ensure the threshold is set below the baseline. |
| Clicking instead of tone | Ground wire loose | Verify Buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (BMP180 reads driving warning alarms) are supported by MbedO interpreted mode.

## Related Projects
- [59 - Pressure Serial](59-pressure-serial.md)
- [61 - Weather Readout LCD](61-weather-readout-lcd.md)
