# 34 - Flame Alarm

Trigger an audible siren alarm immediately when a fire or flame is detected.

## Goal
Learn how to read a digital hazard sensor (Flame Sensor) and trigger an emergency siren wail on a buzzer using conditional checks.

## What You Will Build
When the flame sensor detects a fire, it outputs a digital `LOW` signal (active low). The Arduino detects this and triggers an alternating emergency alarm tone (oscillating between 800 Hz and 1600 Hz) on pin D8. When clear, the buzzer is silenced.

**Why D2 and D8?** Pin D2 reads the digital threshold alarm output from the flame sensor module. Pin D8 drives the wailing alarm buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Flame Sensor | `flame_sensor` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Flame Sensor | VCC | 5V | Power supply (5V) |
| Flame Sensor | DO (Digital Out) | D2 | Digital signal connection |
| Flame Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
const int FLAME_PIN = 2;
const int BUZZ_PIN  = 8;

void setup() {
  pinMode(FLAME_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("Flame Alarm System Ready");
}

void loop() {
  int flameState = digitalRead(FLAME_PIN);
  
  // Active LOW: LOW means flame detected
  if (flameState == LOW) {
    Serial.println("[ALERT] FIRE DETECTED! Sounding Alarm");
    
    // Play wailing siren tone
    tone(BUZZ_PIN, 800);
    delay(150);
    tone(BUZZ_PIN, 1600);
    delay(150);
  } else {
    noTone(BUZZ_PIN);
    delay(100); // Polling delay
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Flame Sensor**, and **Buzzer** onto the canvas.
2. Connect Flame Sensor **VCC** to Arduino **5V**, **DO** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Flame Sensor, adjust the "flame" slider property to trigger the sensor, and listen to the alarm siren.

## Expected Output

Terminal:
```
Flame Alarm System Ready
[ALERT] FIRE DETECTED! Sounding Alarm
[ALERT] FIRE DETECTED! Sounding Alarm
...
```

### Expected Canvas Behavior

| Flame Condition | DO Pin State (D2) | Condition check | Buzzer State (D8) |
| --- | --- | --- | --- |
| Safe (No Flame) | HIGH (5V) | False | LOW - Silent |
| Fire (Flame) | LOW (0V) | True | Oscillating (800/1600 Hz) |

The buzzer vibrates and wails only when the flame slider value crosses the threshold.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int flameState = digitalRead(FLAME_PIN)` | Reads the digital threshold alarm output from the flame sensor module. |
| `if (flameState == LOW)` | Checks if fire is present (active-low comparison). When a flame is detected, it runs the wailing siren loop. |

## Hardware & Safety Concept: Fire Safety Systems
In real fire safety systems, alarms must be highly reliable. Using a digital comparator output allows the microcontrollers to detect hazards instantly with minimal processing overhead. However, optical sensors can be affected by direct sunlight or infrared source reflections, which is why real alarms combine multiple sensor types (such as heat and smoke) to avoid false triggers.

## Try This! (Challenges)
1. **Urgent Wail**: Increase wail speed by changing both `delay(150)` statements to `delay(60)`.
2. **Latch Alarm**: Modify the code so that once the alarm is triggered, it continues to sound indefinitely even if the fire is removed, requiring a button press (or reset) to silence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Siren sounds constantly even when safe | Sensitivity pot set wrong | Adjust the flame slider properties on the canvas to be lower (representing no fire). |
| Click sound instead of wail | Buzzer Pin 2 loose | Verify Buzzer Pin 2 is connected to GND. |

## Mode Notes
These patterns (digital input polling triggering wailing tones) are supported by MbedO interpreted mode.

## Related Projects
- [28 - Flame Detected Print](../beginner/28-flame-detected-print.md)
- [35 - Flame Relay](35-flame-relay.md)
- [116 - Multi-Zone Fire System](../expert/116-multi-zone-fire-system.md)
