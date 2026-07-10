# 44 - Obstacle Beeper

Trigger a repeating backup-alarm alert tone when an obstacle is close to the IR sensor.

## Goal
Learn how to read a digital proximity sensor (IR Sensor) and generate a repeating beep warning pattern on a buzzer when a nearby object is detected.

## What You Will Build
When an object is placed in front of the IR sensor, it pulls pin D2 `LOW`. The Arduino detects this and plays a repeating warning beep (1000 Hz tone for 100 ms, followed by 100 ms of silence) on the buzzer connected to pin D8. When the path is clear, the buzzer remains silent.

**Why D2 and D8?** Pin D2 reads the touchless proximity detection. Pin D8 controls the warning beeper.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| IR Sensor | `ir_sensor` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| IR Sensor | VCC | 5V | Power supply (5V) |
| IR Sensor | OUT | D2 | Digital signal connection |
| IR Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
const int IR_PIN   = 2;
const int BUZZ_PIN = 8;

void setup() {
  pinMode(IR_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("Obstacle Beeper Ready");
}

void loop() {
  int obstacleDetected = digitalRead(IR_PIN);
  
  // Active LOW: LOW means obstacle is detected
  if (obstacleDetected == LOW) {
    Serial.println("[WARNING] Obstacle Close! Beep Alert!");
    
    // Play warning pulse
    tone(BUZZ_PIN, 1000);
    delay(100);
    noTone(BUZZ_PIN);
    delay(100);
  } else {
    // Silent when path is clear
    noTone(BUZZ_PIN);
    delay(100); // Polling delay
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **IR Sensor**, and **Buzzer** onto the canvas.
2. Connect IR Sensor **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the IR Sensor on the canvas to toggle its "Obstacle" property ON, and listen to the repeating beep pattern.

## Expected Output

Terminal:
```
Obstacle Beeper Ready
[WARNING] Obstacle Close! Beep Alert!
[WARNING] Obstacle Close! Beep Alert!
...
```

### Expected Canvas Behavior

| Proximity State | Pin D2 Reading | Buzzer state | Alert Action |
| --- | --- | --- | --- |
| Clear | HIGH (5V) | Silent (0V) | Idle |
| Blocked | LOW (0V) | Beeps at 1000 Hz | Repeating Pulse (100 ms) |

The buzzer vibrates and flashes on the canvas repeatedly while an obstacle is active.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int obstacleDetected = digitalRead(IR_PIN)` | Polls the digital state of the IR sensor. |
| `tone(BUZZ_PIN, 1000); delay(100); noTone(BUZZ_PIN); delay(100);` | Generates a 100 ms pulse followed by 100 ms of silence to form a clear beep alarm loop. |

## Hardware & Safety Concept: Optical Proximity Applications
Proximity alarms of this type are widely used in industrial automation to prevent machinery collisions, and in automotive reversing sensors (parking sonar/radar). The active IR sensor provides a quick, binary indicator that an object has crossed the threshold distance limit.

## Try This! (Challenges)
1. **Urgent Pulse**: Increase the beep speed by reducing both `delay(100)` statements to `delay(40)` (simulating extreme proximity).
2. **Visual indicator**: Wire an LED to D13 and code it to flash in synchronization with the beeper pulses.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Beep loops constantly even when path is clear | Reflection noise or floating wire | Verify that the IR OUT pin is wired securely to pin D2. |
| No sound | Ground missing | Make sure the buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (digital input polling triggering pulsed beep sequences) are supported by MbedO interpreted mode.

## Related Projects
- [22 - Obstacle Print](../beginner/22-obstacle-print.md)
- [38 - Hand Sanitizer Dispenser](38-hand-sanitizer.md)
- [43 - Knock Doorbell](43-knock-doorbell.md)
