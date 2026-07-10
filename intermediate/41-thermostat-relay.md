# 41 - Thermostat Relay

Actuate a heater relay automatically to regulate room temperature using a thermistor.

## Goal
Learn how to read an analog temperature sensor (NTC Thermistor) and implement thermostat hysteresis logic to control a high-power relay.

## What You Will Build
The system monitors ambient temperature. When it gets cold (raw value rises above 600), the relay turns ON to run a heater. When it gets hot (raw value drops below 400), the relay turns OFF to save power.

**Why A0 and D9?** Pin A0 tracks the thermistor voltage. Pin D9 drives the relay controlling the heating element power.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| NTC Thermistor | `ntc_temp` | Yes | Yes |
| Relay Module | `relay_module` | Yes | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Thermistor | VCC | 5V | Power supply (5V) |
| Thermistor | OUT | A0 | Analog signal connection |
| Thermistor | GND | GND | Ground reference |
| Relay Module | VCC | 5V | Power supply (5V) |
| Relay Module | IN | D9 | Digital control line |
| Relay Module | GND | GND | Ground reference |

## Code
```cpp
const int NTC_PIN   = A0;
const int RELAY_PIN = 9;

// Hysteresis threshold limits for NTC (higher value = colder)
const int COLD_LIMIT = 600; // Turn heater ON when cold
const int HOT_LIMIT  = 400; // Turn heater OFF when warm

int heaterActive = LOW;

void setup() {
  pinMode(RELAY_PIN, OUTPUT);
  digitalWrite(RELAY_PIN, heaterActive);
  
  Serial.begin(9600);
  Serial.println("Thermostat Controller Ready");
}

void loop() {
  int tempRaw = analogRead(NTC_PIN);
  
  Serial.print("Temperature Raw: ");
  Serial.print(tempRaw);
  
  // Thermostat regulation logic using NTC behavior
  if (tempRaw > COLD_LIMIT) {
    heaterActive = HIGH;
    Serial.println(" -> Cold detected! Heater ON");
  } 
  else if (tempRaw < HOT_LIMIT) {
    heaterActive = LOW;
    Serial.println(" -> Target reached! Heater OFF");
  } 
  else {
    Serial.print(" -> Regulating. Heater: ");
    Serial.println(heaterActive ? "ON" : "OFF");
  }
  
  digitalWrite(RELAY_PIN, heaterActive);
  delay(500); // Check twice per second
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **NTC Thermistor**, and **Relay Module** onto the canvas.
2. Connect Thermistor **VCC** to Arduino **5V**, **OUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Relay **VCC** to Arduino **5V**, **IN** to Arduino **D9**, and **GND** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Thermistor, adjust the temperature slider below 400 (hotter) and above 600 (colder), and watch the Relay switch.

## Expected Output

Terminal:
```
Thermostat Controller Ready
Temperature Raw: 650 -> Cold detected! Heater ON
Temperature Raw: 500 -> Regulating. Heater: ON
Temperature Raw: 350 -> Target reached! Heater OFF
...
```

### Expected Canvas Behavior

| Temperature State | Pin A0 Reading | heaterActive State | Relay Switch State |
| --- | --- | --- | --- |
| Cold | > 600 | HIGH | ON (Closed) |
| Warm (400 - 600) | 400 - 600 | Retains last state | Retains last state |
| Hot | < 400 | LOW | OFF (Normally Open) |

The relay indicator on the canvas changes state according to the temperature thresholds.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `if (tempRaw > COLD_LIMIT)` | Checks if the room temperature is below the low setpoint. Because this is an NTC thermistor, cold temperatures produce higher raw values. |
| `else if (tempRaw < HOT_LIMIT)` | Deactivates the heater once the room temperature rises above the high setpoint. |

## Hardware & Safety Concept: Thermostat Cycling
If a heating system switched on and off every few seconds, the mechanical contacts in the relay and the furnace elements would burn out quickly. Using **hysteresis** ensures that the heater stays on until the room is fully warm, and then stays off until the temperature has dropped significantly.

## Try This! (Challenges)
1. **Cooling Fan Mode**: Modify the code so that the relay operates a cooling fan instead of a heater (i.e. turn fan ON when hot, and OFF when cold).
2. **Alert indicator**: Wire a Buzzer to D8 and code a single brief warning beep whenever the relay switches states.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Relay oscillates rapidly | Single threshold coded | Verify that you have two distinct limits (`COLD_LIMIT` and `HOT_LIMIT`) in your conditions. |
| Values increase when it gets colder | Normal NTC behavior | This is normal behavior for NTC thermistors. Ensure your thresholds match this inverted scale. |

## Mode Notes
These patterns (analog reads, variable updates, and digital relay output) are supported by MbedO interpreted mode.

## Related Projects
- [20 - Thermistor Raw Print](../beginner/20-thermistor-raw-print.md)
- [40 - Water Pump Control](40-water-pump-control.md)
- [42 - Fan Speed PWM](42-fan-speed-pwm.md)
