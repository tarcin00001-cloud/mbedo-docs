# 110 - ESP32 Soil Moisture Automated Irrigation

Build an automated greenhouse irrigation station that monitors soil moisture levels and switches a relay water valve to irrigate dry soil, implementing a dual-threshold hysteresis loop.

## Goal
Learn how to implement automatic plant watering loops based on soil moisture percentages, using hysteresis control to prevent pump chatter.

## What You Will Build
An analog soil moisture sensor is connected to GPIO 34. A relay controlling a water pump or solenoid valve is connected to GPIO 13. If the soil moisture drops below 30%, the relay turns ON (watering active). The pump continues to run until the moisture level climbs above 60%, then shuts OFF.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Soil Moisture Sensor Module | `soil_sensor` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Soil Sensor | AO | GPIO34 | Yellow | Analog moisture input |
| Soil Sensor | VCC / GND | 3V3 / GND | Red / Black | Power rails |
| Relay Module | IN (Signal) | GPIO13 | Orange | Solenoid valve/pump switch |
| Relay Module | VCC / GND | 5V / GND | Red / Black | Power rails |

> **Wiring tip:** Standard soil moisture sensors output analog voltages. Connect AO to GPIO 34. Power the relay module from the 5V Vin rail to guarantee coil switching current.

## Code
```cpp
// Soil Moisture Automated Irrigation
const int SOIL_PIN = 34;
const int RELAY_PIN = 13;

// Calibration references (raw ADC 0 - 4095)
const int DRY_VAL = 3500; // ADC reading when dry
const int WET_VAL = 1200; // ADC reading when fully wet

// Moisture thresholds in percentage (0% to 100%)
const int MOISTURE_LOW = 30;   // Pump starts watering below this %
const int MOISTURE_HIGH = 60;  // Pump stops watering above this %

bool pumpActive = false;

void setup() {
  Serial.begin(115200);
  
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, LOW); // Start with pump OFF
  
  Serial.println("Automated Irrigation Station Active.");
}

void loop() {
  int rawMoisture = analogRead(SOIL_PIN);
  
  // Map raw values to 0% - 100% moisture range
  int moisturePct = map(rawMoisture, DRY_VAL, WET_VAL, 0, 100);
  moisturePct = constrain(moisturePct, 0, 100);
  
  Serial.print("Moisture: "); Serial.print(moisturePct);
  Serial.print("% (Raw: "); Serial.print(rawMoisture);
  Serial.print(") | Valve: "); Serial.println(pumpActive ? "OPEN (Watering)" : "CLOSED");
  
  // Control Logic
  // 1. Dry soil -> open water valve
  if (moisturePct < MOISTURE_LOW && !pumpActive) {
    Serial.println(">> Soil Dry! Activating Irrigation Valve <<");
    digitalWrite(RELAY_PIN, HIGH);
    pumpActive = true;
  }
  // 2. Wet soil -> close water valve
  else if (moisturePct > MOISTURE_HIGH && pumpActive) {
    Serial.println(">> Soil Wet! Closing Irrigation Valve <<");
    digitalWrite(RELAY_PIN, LOW);
    pumpActive = false;
  }
  
  delay(1000); // Check once per second
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC**, **Soil Moisture Sensor**, and **Relay** onto the canvas.
2. Wire Soil Sensor AO to **GPIO34** and Relay to **GPIO13**.
3. Paste the code and click **Run**.
4. Slide the moisture slider down on the sensor widget (simulating dry soil). Watch the relay turn ON.
5. Slide the moisture slider up (wet soil) past the high threshold. Watch the relay turn OFF.

## Expected Output
Serial Monitor:
```
Automated Irrigation Station Active.
Moisture: 10% (Raw: 3410) | Valve: CLOSED
>> Soil Dry! Activating Irrigation Valve <<
Moisture: 42% (Raw: 2200) | Valve: OPEN (Watering)
Moisture: 72% (Raw: 1150) | Valve: OPEN (Watering)
>> Soil Wet! Closing Irrigation Valve <<
```

## Expected Canvas Behavior
* Starting with a dry sensor slider turns ON the relay (green/active).
* Moving the slider up slowly does not turn off the relay until it crosses the 60% mark.
* Once the slider crosses 60%, the relay shuts off (grey/inactive).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `map(rawMoisture, DRY_VAL, WET_VAL, 0, 100)` | Inverted mapping: lower voltage = higher moisture percentage. |
| `moisturePct < MOISTURE_LOW && !pumpActive` | Starts watering only if the soil is dry and pump is currently OFF. |
| `moisturePct > MOISTURE_HIGH && pumpActive` | Stops watering once the moisture level is satisfied. |

## Hardware & Safety Concept: Sensor Lifespan and Hysteresis
Greenhouse systems check soil levels at long intervals (e.g. once every 10–30 minutes) rather than continuously. Watering plants takes time to distribute through the soil; checking too quickly would cause the pump to run too long, flooding the roots before the water reaches the sensor. Dual-threshold hysteresis prevents the valve from rapidly clicking open/close.

## Try This! (Challenges)
1. **Irrigation HUD**: Integrate a 16x2 I2C LCD displaying the moisture percentage and valve status ("Irrigation: ON").
2. **Water Supply Low Check**: Add a water level sensor (Project 102) in the source tank. Disallow the pump from running if the source tank is empty.
3. **Daily Water Cap Timer**: Keep track of the total time the pump runs during a day. Shut it down if it exceeds 5 minutes total to protect against broken pipes.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks open and close constantly | Soil sensor placement too close to water inlet | Move the sensor further away from the water output line to allow time for absorption |
| Sensor corrosion | Constant current electrolysis | Connect sensor VCC to a GPIO pin and only turn it on when reading |
| Calibration bounds inaccurate | Different soil types | Calibrate `DRY_VAL` and `WET_VAL` in your target soil |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [40 - ESP32 Soil Moisture Analog Level Print](../beginner/40-esp32-soil-moisture-analog-level-print.md)
- [11 - ESP32 Relay Module Control](../beginner/11-esp32-relay-module-control.md)
- [102 - ESP32 Water Level Control Station](102-esp32-water-level-control-station.md)
