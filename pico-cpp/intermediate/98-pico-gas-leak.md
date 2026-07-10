# 98 - Pico Gas Leak

Build a gas leakage safety node that sounds an alarm and triggers a ventilation relay if gas concentration exceeds a threshold.

## Goal
Learn how to use analog gas sensors to trigger simultaneous acoustic alarms and mechanical relay switches for ventilation controls.

## What You Will Build
A gas safety control station:
- **MQ-2 Gas Sensor (GP26)**: Monitors ambient gas/smoke level.
- **Active Buzzer (GP14)**: Sounds a rapid alert siren if the gas level exceeds the threshold.
- **Relay Module (GP10)**: Activates a 5V exhaust fan relay when gas levels exceed the threshold to ventilate the area.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| Relay Module | `relay` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Heater power |
| MQ-2 Sensor | AO | GP26 | Analog input |
| MQ-2 Sensor | GND | GND | Ground reference |
| Active Buzzer | VCC (+) | GP14 | Sounder control |
| Active Buzzer | GND (-) | GND | Ground return |
| Relay Module | VCC | 5V | Coil power |
| Relay Module | IN | GP10 | Exhaust fan control |
| Relay Module | GND | GND | Ground return |

## Code
```cpp
const int MQ2_PIN    = 26;
const int BUZZER_PIN = 14;
const int RELAY_PIN  = 10;

// Gas alarm limit (raw analog reading)
const int GAS_LIMIT = 1800;

void setup() {
  pinMode(MQ2_PIN, INPUT);
  pinMode(BUZZER_PIN, OUTPUT);
  pinMode(RELAY_PIN, OUTPUT);
  
  digitalWrite(BUZZER_PIN, LOW);
  digitalWrite(RELAY_PIN, LOW);
}

void loop() {
  int gasLevel = analogRead(MQ2_PIN);

  // If gas level exceeds the safety limit
  if (gasLevel > GAS_LIMIT) {
    digitalWrite(RELAY_PIN, HIGH); // Turn exhaust fan ON
    
    // Pulse alarm siren
    digitalWrite(BUZZER_PIN, HIGH);
    delay(150);
    digitalWrite(BUZZER_PIN, LOW);
    delay(150);
  } else {
    digitalWrite(RELAY_PIN, LOW);  // Fan OFF
    digitalWrite(BUZZER_PIN, LOW); // Siren OFF
  }

  delay(200); // Check 5 times per second
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **MQ-2 Gas Sensor**, **Active Buzzer**, and **Relay Module** onto the canvas.
2. Connect Gas Sensor to **GP26**, Buzzer to **GP14**, and Relay to **GP10**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Adjust the gas PPM slider above 1800 and check that the buzzer pulses and the relay clicks ON.

## Expected Output

Terminal:
```
Simulation active. Gas monitor checking GP26. Threshold: 1800.
```

## Expected Canvas Behavior
| Gas PPM Slider level | GP14 (Buzzer) State | GP10 (Relay) State | Safety Status |
| --- | --- | --- | --- |
| < 1800 | LOW | LOW | Normal |
| > 1800 | Pulsing HIGH/LOW | HIGH (Closed) | **GAS LEAK ALARM ACTIVE** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(RELAY_PIN, HIGH)` | Drives GP10 HIGH, closing the relay contacts to activate the ventilation fan. |

## Hardware & Safety Concept: Heater Current Requirements
MQ-series gas sensors contain an internal heater element that requires a constant current of around 150mA. This heating power is necessary to keep the metal-oxide sensing layer at its operating temperature (around 300°C). Attempting to power an MQ sensor from the Pico's 3.3V pin will overload the regulator and drop the rail voltage, causing resets. Always power MQ sensor heater pins from the Pico's 5V pin (VBUS) or an external supply.

## Try This! (Challenges)
1. **Warning LED**: Add a Red LED on GP15 and flash it in sync with the buzzer alarm.
2. **Latch Alert**: Modify the code so that if a gas leak is detected, the alarm stays active until a reset button (GP17) is pressed.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Alarm triggers immediately on boot | Sensor warm-up phase | MQ sensors require a brief warm-up period (1-2 minutes) to reach stable operating temperatures. Ignore values for the first 30 seconds after startup. |

## Mode Notes
This multi-device analog control project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [11 - Pico Relay Toggle](../../beginner/11-pico-relay-toggle.md)
- [43 - Pico MQ-2 Gas Sensor](../../beginner/43-pico-mq2-gas.md)
