# 32 - Gas Alarm

Trigger an audible alarm siren when gas or smoke levels exceed a threshold limit.

## Goal
Learn how to read an analog gas sensor (MQ-2) and trigger an audible buzzer alarm using comparison threshold checks.

## What You Will Build
When the MQ-2 gas concentration reading exceeds 500 (representing hazardous smoke or gas levels), the buzzer connected to pin D8 outputs a fast warning beep. When the level drops below 500, the buzzer is silenced.

**Why A0 and D8?** Pin A0 tracks the continuous gas concentration levels. Pin D8 controls the alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| MQ-2 Gas Sensor | `gas_sensor` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| MQ-2 Sensor | VCC | 5V | Power supply (5V) |
| MQ-2 Sensor | AOUT (Analog Out) | A0 | Analog signal connection |
| MQ-2 Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
const int GAS_PIN  = A0;
const int BUZZ_PIN = 8;

// Threshold level for gas alert (0 to 1023)
const int GAS_THRESHOLD = 500;

void setup() {
  pinMode(BUZZ_PIN, OUTPUT);
  Serial.begin(9600);
  Serial.println("Gas Alarm Monitor Ready");
  delay(1000); // Warm up delay
}

void loop() {
  int gasLevel = analogRead(GAS_PIN);
  
  Serial.print("Gas Level: ");
  Serial.println(gasLevel);
  
  // Trigger alarm if level exceeds threshold
  if (gasLevel > GAS_THRESHOLD) {
    Serial.println("[WARNING] Gas concentration high! Siren active");
    
    // Play fast beep wail
    tone(BUZZ_PIN, 1000);
    delay(100);
    tone(BUZZ_PIN, 2000);
    delay(100);
  } else {
    noTone(BUZZ_PIN);
    delay(200); // Poll rate delay
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **MQ2 Gas Sensor**, and **Buzzer** onto the canvas.
2. Connect MQ-2 **VCC** to Arduino **5V**, **AOUT** to Arduino **A0**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the MQ-2 sensor on the canvas, adjust the gas slider to exceed 500, and watch the alarm trigger.

## Expected Output

Terminal:
```
Gas Alarm Monitor Ready
Gas Level: 120
Gas Level: 580
[WARNING] Gas concentration high! Siren active
...
```

### Expected Canvas Behavior

| Gas concentration | Pin A0 Reading | Condition check | Buzzer State (D8) |
| --- | --- | --- | --- |
| Normal (< 500) | < 500 | False | LOW - Silent |
| Hazardous (> 500) | > 500 | True | Oscillating (1000/2000 Hz) |

The buzzer vibrates and wails only when the slider concentration goes above 500.

## Code Walkthrough

| Line / Expression | What It Does |
| --- | --- |
| `int gasLevel = analogRead(GAS_PIN)` | Reads the analog output voltage from the MQ-2 sensor. |
| `if (gasLevel > GAS_THRESHOLD)` | Compares the current reading against our threshold level of 500 to determine if the environment is safe. |

## Hardware & Safety Concept: Gas Leak Detection
Gas sensor modules (like MQ-2, MQ-3, or MQ-7) contain a heating element that is required to reach a specific operating temperature to react with incoming gases. Because of this, these sensors consume substantial current (approx. 150 mA) and can run hot to the touch. Always ensure your power supply can handle this current load.

## Try This! (Challenges)
1. **Adjust Sensitivity**: Change the `GAS_THRESHOLD` in code to `300` to make the alarm trigger earlier.
2. **Visual Alert**: Connect an LED to D13 and modify the code so the LED flashes in sync with the siren.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Siren triggers constantly even in clean air | Threshold set too low | Check the raw values printed in the Terminal, and raise `GAS_THRESHOLD` above the ambient level. |
| No sound | Ground missing | Make sure the buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (analog sensor polling triggering wailing tones) are supported by MbedO interpreted mode.

## Related Projects
- [29 - Gas Level Print](../beginner/29-gas-level-print.md)
- [33 - Gas LED Alert](33-gas-led-alert.md)
- [87 - Air Quality Monitor](../advanced/87-air-quality-monitor.md)
