# 44 - Hall Effect Magnetic Sensor Print (GPIO Analog pin)

Measure and print magnetic field strength and polarity using an analog Hall Effect sensor on the VEGA ARIES v3 board.

## Goal
Understand the physics of the Hall Effect, wire an analog Hall sensor to the ARIES board, and write code to calculate magnetic pole direction and relative magnetic field intensity.

## What You Will Build
An analog Hall Effect sensor is connected to analog input pin `ADC0` (`GP26`). In the absence of a magnetic field, the sensor outputs a midpoint voltage (~1.65V). When a magnet is brought close, the voltage shifts up or down depending on the pole. The board reads this voltage, calculates the field offset, and prints the strength and polarity.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Analog Hall Effect Sensor | `hall_sensor` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Hall Sensor | VCC | 3.3V | Red | Power connection |
| Hall Sensor | AO (Analog Out) | ADC0 (GP26) | Yellow | Analog magnetic strength signal |
| Hall Sensor | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use the Analog Output (AO) pin from the sensor module. When no magnet is near, the voltage sits at approximately 1.65V (half of VCC), which corresponds to a raw reading of ~2048.

## Code
```cpp
// Hall Effect Magnetic Sensor Print - VEGA ARIES v3
const int HALL_PIN = GP26; // ADC0

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw 12-bit analog value from the Hall sensor
  int rawValue = analogRead(HALL_PIN);
  
  // Calculate relative magnetic field offset from nominal midpoint (2048)
  // Positive values represent South Pole; Negative values represent North Pole
  int magneticField = rawValue - 2048;
  
  // Print raw value and calculated field strength/direction
  Serial.print("Raw Hall: ");
  Serial.print(rawValue);
  Serial.print(" | Magnetic Field Strength/Direction: ");
  Serial.println(magneticField);
  
  // Update every 500 milliseconds
  delay(500);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and an **Analog Hall Effect Sensor** onto the canvas.
2. Wire the Hall Sensor's VCC to **3.3V**, AO to **GP26 (ADC0)**, and GND to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Bring a simulated magnet close to the sensor on the canvas (using the slider) and monitor the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Raw Hall: 2048 | Magnetic Field Strength/Direction: 0
Raw Hall: 3100 | Magnetic Field Strength/Direction: 1052
Raw Hall: 950 | Magnetic Field Strength/Direction: -1098
```

## Expected Canvas Behavior
* Adjusting the magnetic field slider on the Hall Effect widget displays values that increase (South Pole) or decrease (North Pole) relative to the 2048 zero-point in the virtual terminal.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int HALL_PIN = GP26;` | Defines the analog pin for the Hall Effect sensor. |
| `analogRead(HALL_PIN)` | Reads the 12-bit ADC value. |
| `rawValue - 2048` | Offsets the raw value by the nominal 1.65V zero-field midpoint to get the magnitude and polarity (+ for South, - for North). |

## Hardware & Safety Concept: Hall Effect and Magnetic Pole Sensitivity
* **Hall Effect**: The Hall Effect is the production of a voltage difference (the Hall voltage) across an electrical conductor, transverse to an electric current in the conductor and to an applied magnetic field perpendicular to the current.
* **Magnetic Pole Sensitivity**: Linear (analog) Hall Effect sensors can distinguish between North and South magnetic poles. One pole (e.g. South) increases the charge carriers' deflection, raising the voltage above VCC/2. The opposite pole (North) deflects them in the reverse direction, lowering the output voltage towards 0V.

## Try This! (Challenges)
1. **Magnetic Pole Indicator**: Turn ON the onboard Red LED (`LED_R`) if a strong North pole is detected (negative field) and the onboard Blue LED (`LED_B`) if a strong South pole is detected (positive field).
2. **Proximity Alarm**: Connect a buzzer to `GPIO 14`. Sound the buzzer with increasing beep rates as a magnetic source comes closer (absolute value of `magneticField` increases).

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Midpoint is not exactly 2048 | Minor sensor tolerances or Earth's magnetic field | This is normal on hardware. Write a baseline calibration step in `setup()` to record the zero-field average, and use that value instead of the hardcoded 2048 |
| Reading does not change when magnet is brought close | Digital Hall sensor used instead of analog | Verify if you are using a digital Hall sensor (e.g. US1881, which outputs HIGH/LOW latched values) or an analog Hall sensor (e.g. AH3503 or SS49E) |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [43 - Flame sensor digital alarm](43-flame-sensor-digital-alarm.md) (Previous project)
- [45 - Potentiometer servo sweep](45-potentiometer-servo-sweep.md) (Next project)
