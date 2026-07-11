# 40 - Soil Moisture Sensor Level Print (GPIO Analog pin)

Read and display soil moisture percentage values using a soil moisture sensor connected to the VEGA ARIES v3 board.

## Goal
Understand how resistive soil moisture sensors function, wire an analog sensor module to the ARIES board, and write code to map raw ADC readings to a 0–100% moisture scale.

## What You Will Build
A resistive soil moisture sensor is placed in soil and wired to analog input pin `ADC3` (`GP29`). The board reads the changing voltage caused by soil conductivity, maps it from a raw reading (4095 for completely dry to 0 for fully wet) to a percentage (0% to 100%), and prints it to the Serial Monitor.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Soil Moisture Sensor & Module | `soil_moisture` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Moisture Sensor | VCC | 3.3V | Red | Power connection |
| Soil Moisture Sensor | AO (Analog Out) | ADC3 (GP29) | Yellow | Analog moisture level signal |
| Soil Moisture Sensor | GND | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Use the Analog Output (AO) pin from the driver board to GP29 (ADC3) to get a granular moisture reading. The Digital Output (DO) threshold pin on the driver board can remain disconnected.

## Code
```cpp
// Soil Moisture Sensor Level Print - VEGA ARIES v3
const int SOIL_PIN = GP29; // ADC3

void setup() {
  // Initialize serial communication at 115200 bps
  Serial.begin(115200);
}

void loop() {
  // Read the raw analog value from the sensor
  int rawValue = analogRead(SOIL_PIN);
  
  // Map raw value: Dry (4095) is 0%, fully wet (0) is 100% moisture
  int moisturePercent = map(rawValue, 4095, 0, 0, 100);
  
  // Clamp values between 0 and 100 to filter electrical noise
  if (moisturePercent < 0) {
    moisturePercent = 0;
  }
  if (moisturePercent > 100) {
    moisturePercent = 100;
  }
  
  // Print values to the Serial Monitor
  Serial.print("Soil ADC Raw: ");
  Serial.print(rawValue);
  Serial.print(" | Moisture: ");
  Serial.print(moisturePercent);
  Serial.println("%");
  
  // Wait 1 second before the next measurement
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and a **Soil Moisture Sensor** onto the canvas.
2. Wire the Soil Moisture Sensor's VCC to **3.3V**, AO to **GP29 (ADC3)**, and GND to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Adjust the moisture level slider on the soil moisture sensor widget and watch the Serial Monitor.

## Expected Output
Serial Monitor:
```
System Initialized.
Soil ADC Raw: 4095 | Moisture: 0%
Soil ADC Raw: 1638 | Moisture: 60%
Soil ADC Raw: 0 | Moisture: 100%
```

## Expected Canvas Behavior
* The Serial Monitor prints the mapped moisture level from 0% (bone dry) to 100% (saturated soil) as the soil moisture sensor slider is moved.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int SOIL_PIN = GP29;` | Maps `GP29` (ADC3) as the analog soil moisture sensor input pin. |
| `map(rawValue, 4095, 0, 0, 100)` | Maps the dry value (high resistance/voltage) to 0% and wet value (low resistance/voltage) to 100%. |
| `if (moisturePercent < 0) ...` | Clamps moisture values to prevent negative numbers or numbers exceeding 100. |

## Hardware & Safety Concept: Resistive vs. Capacitive Sensors and Corrosion
* **Resistive Sensors**: Resistive soil moisture probes measure the resistance of the soil. Dry soil has very low conductivity (high resistance). Water and dissolved nutrients increase soil conductivity, dropping the resistance.
* **Corrosion**: Resistive probes are highly prone to corrosion because direct DC current causes electro-chemical plating on the probes, destroying the metal plating over time. Capacitive soil moisture sensors are preferred for long-term deployments because their electrodes are insulated from the soil.

## Try This! (Challenges)
1. **Automated Irrigation Alert**: Turn ON the onboard Warning LED (`GPIO 15`) to indicate a "Dry Soil" alarm if moisture falls below 30%.
2. **Sensor Duty Cycle**: Connect the VCC of the sensor module to a digital pin (like `GPIO 15`) and turn it HIGH only during a reading to minimize probe wear and corrosion.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Moisture value stays at 0% even in wet soil | Open circuit or bad connection | Ensure the jumper wires between the sensor probes and the amplifier module are firmly connected |
| Erratic jumps in readings | Poor soil contact or air gaps | Press the soil firmly around the probes to remove air gaps that block conductivity |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [39 - Rain sensor analog read](39-rain-sensor-analog-read.md) (Previous project)
- [41 - Water level sensor level print](41-water-level-sensor-level-print.md) (Next project)
