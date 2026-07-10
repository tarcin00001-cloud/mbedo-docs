# 40 - Water Pump Control

Actuate a water pump relay automatically to refill a tank based on liquid levels.

## Goal
Learn how to implement a basic hysteresis pump controller using an analog water level sensor and a high-power relay module.

## What You Will Build
The system monitors water depth. When water level drops below 30% (raw reading 300), the relay turns ON to start refilling. When level rises above 80% (raw reading 800), the relay turns OFF to prevent overflowing.

**Why A0 and D9?** Pin A0 measures depth analog voltage. Pin D9 drives the relay controlling the pump power.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Water Level Sensor | `water_level_sensor` | Yes | Yes |
| Relay Module | `relay_module` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Water Level Sensor | VCC | 5V | Power supply (5V) |
| Water Level Sensor | SIG | A0 | Analog signal connection |
| Water Level Sensor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply (5V) |
| Relay Module | IN | D9 | Digital control line |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int WATER_PIN = A0;
const int RELAY_PIN = 9;

// Hysteresis threshold limits (0 to 1023)
const int LOW_THRESHOLD  = 300; // Refill start limit (approx. 30%)
const int HIGH_THRESHOLD = 800; // Refill stop limit (approx. 80%)

int pumpActive = LOW; // Track the current pump state

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, pumpActive);
  
  Serial.begin(9600);
  Serial.println("Water Pump Controller Ready");
}

void loop() {
  int level = analogRead(WATER_PIN);
  
  Serial.print("Water Raw Value: ");
  Serial.print(level);
  
  // Refill logic using hysteresis
  if (level < LOW_THRESHOLD) {
    pumpActive = HIGH;
    Serial.println(" -> Tank empty! Refill started (Pump ON)");
  } 
  else if (level > HIGH_THRESHOLD) {
    pumpActive = LOW;
    Serial.println(" -> Tank full! Refill complete (Pump OFF)");
  } 
  else {
    Serial.print(" -> Level OK. Pump state: ");
    Serial.println(pumpActive ? "ON" : "OFF");
  }
  
  digitalWrite(RELAY_PIN, pumpActive);
  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Water Level Sensor**, and **Relay Module** onto the canvas.
2. Connect Water Level Sensor **VCC** to Arduino **5V**, **SIG** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Relay **VCC** to Arduino **5V**, **IN** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Water Level Sensor, adjust the level slider below 300 to start refilling, and watch it stop when you raise it above 800.

## Expected Output

Terminal:
```
Water Pump Controller Ready
Water Raw Value: 200 -> Tank empty! Refill started (Pump ON)
Water Raw Value: 500 -> Level OK. Pump state: ON
Water Raw Value: 850 -> Tank full! Refill complete (Pump OFF)
...
```

### Expected Canvas Behavior

| Water Depth Slider | Pin A0 Reading | pumpActive State | Relay Switch State |
| --- | --- | --- | --- |
| Low (< 300) | < 300 | HIGH | ON (Closed) |
| Filling (300 - 800) | 300 - 800 | Retains last state | Retains last state |
| High (> 800) | > 800 | LOW | OFF (Normally Open) |

The pump stays active until the high-limit is crossed, and stays off until the low-limit is crossed.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (level < LOW_THRESHOLD)` | Triggers refilling by setting `pumpActive = HIGH`. |
| `else if (level > HIGH_THRESHOLD)` | Stops refilling by setting `pumpActive = LOW`. |
| `digitalWrite(RELAY_PIN, pumpActive)` | Actuates the physical relay control line based on the state variable. |

## Hardware & Safety Concept: Hysteresis Controller
If you set a single control point (e.g. turn pump ON if level < 500 and OFF if > 500), water ripples and wave noise on the surface would cause the relay to click on and off rapidly when hovering at 500. By splitting this into two different limits (**Hysteresis**), we ensure the pump runs smoothly for long cycles, preventing wear and protecting the electrical contacts.

## Try This! (Challenges)
1. **Critical Empty Alarm**: Connect a Buzzer to D8 and code a siren alarm that triggers if the water level falls below `100` (representing empty dry tank danger).
2. **Refill Timeout Safe**: Prevent infinite refilling (e.g., if a pipe leaks). Add a timer counter that stops the pump if it runs for more than 15 loops without reaching the high limit.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay clicks repeatedly at the same limit | Single threshold coded | Verify that you have two distinct limits (`LOW_THRESHOLD` and `HIGH_THRESHOLD`) in your conditions. |
| Water depth changes but raw value stays stuck | Sensor pin mismatch | Ensure the water level sensor `SIG` connects to A0. |

## Mode Notes
These patterns (analog reads, logic bounds, and digital outputs) are supported by MbedO interpreted mode.

## Related Projects
- [26 - Water Level Print](../beginner/26-water-level-print.md)
- [37 - Night Fan Relay](37-night-fan-relay.md)
- [89 - Flood Alert](../advanced/89-flood-alert.md)
