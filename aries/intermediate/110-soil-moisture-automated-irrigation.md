# 110 - Soil Moisture Automated Irrigation

Build an automated irrigation console that reads soil moisture levels using an analog soil sensor and controls a watering pump via a 5V relay module on the VEGA ARIES v3 board.

## Goal
Learn how to interface analog resistive soil moisture sensors, construct a hysteresis control loop for soil watering, and operate relays to control water valves.

## What You Will Build
An automatic plant watering system. The system measures soil moisture. If the soil becomes dry (analog value exceeds 2500), the relay turns ON to drive a water pump or solenoid valve. The pump stays active until the soil is watered and moisture levels increase (value falls below 1500), preventing short-cycling.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Soil Moisture Sensor | `soil_moisture` | Yes | Yes |
| 5V Relay Module (Water Pump) | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | VCC | 3V3 | Red | Sensor operating voltage (3.3V) |
| Soil Sensor | GND | GND | Black | Ground reference |
| Soil Sensor | SIG (Signal) | ADC3 (GP29) | White | Analog moisture output |
| Relay Module | VCC | 5V | Red | Relay operating voltage (5V) |
| Relay Module | GND | GND | Black | Ground reference |
| Relay Module | IN | GPIO 15 | Blue | Control signal pin |

> **Wiring tip:** Soil moisture sensors have probe forks inserted in the soil and a signal conditioner board. Power the signal conditioner board from 3.3V to match the ARIES ADC operating limits.

## Code
```cpp
const int SOIL_PIN = 29;    // Soil moisture sensor on ADC3 (GP29)
const int RELAY_PIN = 15;   // Solenoid / Pump Relay on GPIO 15

// Thresholds for moisture control (lower values = wetter soil)
const int DRY_THRESHOLD = 2500; // Trigger watering above this value
const int WET_THRESHOLD = 1500; // Stop watering below this value

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with pump OFF
}

void loop() {
  int moisture = analogRead(SOIL_PIN);

  // Hysteresis loop to prevent relay chattering
  if (moisture > DRY_THRESHOLD) {
    digitalWrite(RELAY_PIN, HIGH); // Turn pump ON (watering)
  } else if (moisture < WET_THRESHOLD) {
    digitalWrite(RELAY_PIN, LOW);  // Turn pump OFF (stop watering)
  }

  delay(500); // Poll moisture levels twice per second
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **Soil Moisture Sensor**, and **5V Relay Module** components onto the canvas.
2. Wire the Soil Sensor: **VCC** to **3V3**, **GND** to **GND**, and **SIG** to **ADC3 (GP29)**.
3. Wire the Relay: **VCC** to **5V**, **GND** to **GND**, and **IN** to **GPIO 15**.
4. Paste the code into the editor.
5. Select **Interpreted Mode** in the simulation dropdown.
6. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Soil Moisture ADC: 3100 (Dry) -> PUMP: ACTIVE
Soil Moisture ADC: 2100 (Moist) -> PUMP: ACTIVE
Soil Moisture ADC: 1400 (Wet) -> PUMP: INACTIVE
```

## Expected Canvas Behavior
* Increasing the simulated soil dryness slider (increasing the analog value above 2500) turns on the relay.
* Decreasing the dryness slider below 1500 turns off the relay.
* Keeping the dryness slider within the 1500-2500 band maintains the current state of the relay pump.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `analogRead(SOIL_PIN)` | Reads the analog moisture level (0 for wet probe, 4095 for dry air). |
| `moisture > DRY_THRESHOLD` | Evaluates if the soil has dried out past the dry setpoint. |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GPIO 15 high to close the relay contacts. |
| `moisture < WET_THRESHOLD` | Shuts down irrigation once soil is sufficiently wet. |

## Hardware & Safety Concept
* **Solenoid Valve Current**: 12V water pumps or solenoids require external power. When wiring, route the external power through the relay Common (COM) and Normally Open (NO) terminals, and keep it physically isolated from ARIES digital lines.
* **Corrosion Resistance**: Cheap resistive soil probes corrode quickly in wet soil due to electrolysis. For long-term hardware builds, use capacitive soil moisture sensors, which are coated in plastic and measure moisture using electrical capacitance instead of direct current resistance.

## Try This! (Challenges)
1. **Status Warning LED**: Connect a warning LED to GPIO 14. If the soil moisture remains above the dry threshold for more than 10 seconds (indicating the water tank is empty or the pump failed), blink the warning LED rapidly.
2. **Irrigation Duty Cycle**: To prevent overwatering, restrict watering cycles so the pump runs for a maximum of 3 seconds, followed by a mandatory 10-second wait to allow water to soak into the soil.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Soil reading stays at 4095 | Sensor probe disconnected | Verify that the wire leads between the probe fork and signal conditioner are connected. |
| Relay switches on/off rapidly | Hysteresis thresholds are too close | Expand the gap between the `DRY_THRESHOLD` and `WET_THRESHOLD` values. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [102 - Water Level Control Station](102-water-level-control-station.md)
- [104 - Night Light Controller](104-night-light-controller.md)
