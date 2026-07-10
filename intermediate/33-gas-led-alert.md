# 33 - Gas LED Alert

Provide visual safety alerts using an LED when gas levels exceed a threshold.

## Goal
Learn how to read an analog input sensor (MQ-2) and drive a digital output indicator (LED) using conditional comparison checks.

## What You Will Build
When the gas concentration level measured by the MQ-2 sensor exceeds 500, the warning LED on D13 turns ON. When levels are safe, the LED turns OFF.

**Why A0 and D13?** Pin A0 tracks the analog gas level. Pin D13 drives our visual warning indicator.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| LED | `led` | Yes | Yes |
| 220 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Power supply (5V) |
| MQ-2 Sensor | AOUT | A0 | Analog signal connection |
| MQ-2 Sensor | GND | GND | Ground reference |
| LED | A (Anode, longer leg) | D13 | Output signal connection |
| LED | C (Cathode, shorter leg) | GND | Ground reference |

## Code
```cpp
const int GAS_PIN = A0;
const int LED_PIN = 13;

const int GAS_THRESHOLD = 500;

void setup() {
  pinMode(LED_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Gas LED Monitor Ready");
}

void loop() {
  int gasLevel = analogRead(GAS_PIN);
  
  Serial.print("Gas: ");
  Serial.println(gasLevel);
  
  if (gasLevel > GAS_THRESHOLD) {
    digitalWrite(LED_PIN, HIGH);
    Serial.println("WARNING - Gas limit exceeded! LED ON");
  } else {
    digitalWrite(LED_PIN, LOW);
  }
  
  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MQ2 Gas Sensor**, and **LED** onto the canvas.
2. Connect MQ-2 **VCC** to Arduino **5V**, **AOUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect LED **A** (Anode) to Arduino **D13** and LED **C** (Cathode) to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the MQ-2 sensor, adjust the gas concentration slider to exceed 500, and watch the LED light up.

## Expected Output

Terminal:
```
Gas LED Monitor Ready
Gas: 150
Gas: 620
WARNING - Gas limit exceeded! LED ON
...
```

### Expected Canvas Behavior

| Gas Concentration | Pin A0 Reading | Condition check | LED State (D13) |
| --- | --- | --- | --- |
| Safe | < 500 | False | LOW - OFF |
| Danger | > 500 | True | HIGH - ON |

The LED on the canvas lights up immediately when the gas slider passes 500.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int gasLevel = analogRead(GAS_PIN)` | Reads the analog output voltage from the MQ-2 sensor. |
| `if (gasLevel > GAS_THRESHOLD)` | Evaluates if the air concentration exceeds our safety limit. If true, the LED pin is driven HIGH. |

## Hardware & Safety Concept: Threshold Selection
Different gases react with the sensing element in different ways. For example, the MQ-2 is highly sensitive to LPG and propane, but less sensitive to hydrogen. Choosing the correct threshold in code is critical in real-world systems to prevent false alarms while ensuring hazardous conditions are detected.

## Try This! (Challenges)
1. **Pulse Alert**: Make the LED flash on and off repeatedly (`HIGH` -> delay -> `LOW` -> delay) when in the danger zone instead of staying solid.
2. **Reverse Safety Indicator**: Modify the code so the LED behaves as a "Safe" light: it remains ON normally, and turns OFF only when gas is detected.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| LED stays on constantly | Threshold too low or floating wire | Read the raw values printed in the Terminal, and raise `GAS_THRESHOLD` above the ambient level. |
| LED never turns on | Polarized LED reversed | Swap wires: Anode (A) to D13, Cathode (C) to GND. |

## Mode Notes
These patterns (analog sensor polling triggering digital outputs) are supported by MbedO interpreted mode.

## Related Projects
- [29 - Gas Level Print](../beginner/29-gas-level-print.md)
- [32 - Gas Alarm](32-gas-alarm.md)
- [87 - Air Quality Monitor](../advanced/87-air-quality-monitor.md)
