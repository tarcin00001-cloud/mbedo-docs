# 31 - Motion Alarm

Trigger an audible siren alarm when motion is detected by a PIR sensor.

## Goal
Learn how to use a digital input (PIR motion sensor) to trigger an oscillating audio output (siren tone) using conditional checks.

## What You Will Build
When the PIR sensor detects motion, the buzzer connected to pin D8 outputs an alternating siren tone (alternating between 600 Hz and 1200 Hz). When quiet, the buzzer is silenced.

**Why D2 and D8?** Pin D2 reads the digital output of the PIR sensor. Pin D8 acts as our audio signal line.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| PIR Motion Sensor | `pir` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| PIR Sensor | VCC | 5V | Power supply (5V) |
| PIR Sensor | OUT | D2 | Digital signal connection |
| PIR Sensor | GND | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

## Code
```cpp
const int PIR_PIN  = 2;
const int BUZZ_PIN = 8;

void setup() {
  pinMode(PIR_PIN, INPUT);
  Serial.begin(9600);
  Serial.println("Motion Alarm System Ready");
}

void loop() {
  int motionDetected = digitalRead(PIR_PIN);
  
  if (motionDetected == HIGH) {
    Serial.println("[ALERT] Intruder Detected! Sounding Alarm");
    
    // Play siren tone (oscillates between two frequencies)
    tone(BUZZ_PIN, 600);
    delay(200);
    tone(BUZZ_PIN, 1200);
    delay(200);
  } else {
    noTone(BUZZ_PIN);
    delay(100);
  }
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **PIR Motion Sensor**, and **Buzzer** onto the canvas.
2. Connect PIR **VCC** to Arduino **5V**, **OUT** to Arduino **D2**, and **GND** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Click the PIR sensor on the canvas to toggle its "Motion" property ON and OFF, and listen to the siren response.

## Expected Output

Terminal:
```
Motion Alarm System Ready
[ALERT] Intruder Detected! Sounding Alarm
[ALERT] Intruder Detected! Sounding Alarm
...
```

### Expected Canvas Behavior

| PIR Sensor State | Pin D2 Reading | Buzzer Pin D8 State | Sound Output |
| --- | --- | --- | --- |
| Quiet (No Motion) | LOW (0V) | Inactive (0V) | Silent |
| Active (Motion) | HIGH (5V) | Oscillates (600/1200 Hz) | Emergency Siren |

The buzzer oscillates and vibrates on the canvas only when motion is detected.

## Code Walkthrough

| Block / Statement | What It Does |
| --- | --- |
| `if (motionDetected == HIGH)` | Triggers the siren tone wail loop if the PIR output is HIGH. |
| `tone(BUZZ_PIN, 600); delay(200);` | Generates a 600 Hz pitch, holds for 200 ms, then immediately overrides it with 1200 Hz for another 200 ms. |
| `noTone(BUZZ_PIN)` | Silences the buzzer immediately when the motion pin drops to LOW. |

## Hardware & Safety Concept: Siren Design
Emergency sirens use frequency modulation (changing pitch over time) rather than a single continuous tone because alternating frequencies are much more noticeable to human hearing and stand out against background environment noise.

## Try This! (Challenges)
1. **Urgent Siren**: Speed up the alarm wail by changing both `delay(200)` statements to `delay(80)`.
2. **Delayed Off**: Modify the alarm so that once triggered, the alarm sounds for at least 5 seconds (5000 ms) before checking if the area is safe again.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Siren plays once and then the program freezes | Missing delay statement | Check that you have `delay(100)` in the `else` block to prevent loop locking. |
| Buzzer click but no sound | Ground connection missing | Verify Buzzer Pin 2 is connected directly to GND. |

## Mode Notes
These patterns (digital input polling triggering wailing tone outputs) are supported by MbedO interpreted mode.

## Related Projects
- [10 - Alarm Oscillate](../beginner/10-alarm-oscillate.md)
- [30 - Motion Light](30-motion-light.md)
- [95 - BT Alarm Arm/Disarm](../advanced/95-bt-alarm-arm-disarm.md)
