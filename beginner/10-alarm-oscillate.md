# 10 - Alarm Oscillate

Produce a wailing siren sound by alternating between two frequencies rapidly.

## Goal
Learn how to create a continuous siren effect by switching the output frequency of a buzzer repeatedly.

## What You Will Build
A buzzer connected to pin D8 alternates between a low tone (500 Hz) and a high tone (1500 Hz) every 300 ms to simulate an emergency alarm siren.

**Why D8?** Keeping the sound generation on D8 aligns with all beginner audio projects.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes |

## Wiring
| Buzzer Pin | Arduino Pin | Notes |
| --- | --- | --- |
| Pin 1 (Positive / Signal) | D8 | Audio control line |
| Pin 2 (Negative / Ground) | GND | Ground connection |

## Code
```cpp
const int BUZZ_PIN = 8;

void setup() {
  Serial.begin(9600);
  Serial.println("Alarm Oscillate ready");
}

void loop() {
  // Play low-pitch wail (500 Hz)
  tone(BUZZ_PIN, 500);
  delay(300);
  
  // Play high-pitch wail (1500 Hz)
  tone(BUZZ_PIN, 1500);
  delay(300);
}
```

## What to Click in MbedO
1. Drag **Arduino Uno** onto the canvas.
2. Drag **Buzzer** onto the canvas.
3. Connect Buzzer **pin 1** to Arduino **D8**.
4. Connect Buzzer **pin 2** to Arduino **GND**.
5. Paste the code into the editor.
6. Use the default Arduino interpreted runtime.
7. Click **Run**.

## Expected Output

Serial Monitor:
```
Alarm Oscillate ready
```

### Expected Canvas Behavior

| Loop Step | Tone Pitch | Duration | Sound Character |
| --- | --- | --- | --- |
| 1 | 500 Hz | 300 ms | Low pitch |
| 2 | 1500 Hz | 300 ms | High pitch |
| Repeats | Cycles | 600 ms total | Wailing siren |

The buzzer vibrates and flashes on the canvas continuously.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `tone(BUZZ_PIN, 500)` | Generates a 500 Hz tone on D8. |
| `delay(300)` | Maintains the 500 Hz tone for 300 ms. |
| `tone(BUZZ_PIN, 1500)` | Immediately switches the output frequency on D8 to 1500 Hz. (No `noTone` is needed here because the new frequency overrides the old one). |

## Hardware & Safety Concept: Frequency Sweeps
In professional sirens, the tone doesn't jump instantly between two pitches (which is a **dual-tone alarm**). Instead, it sweeps continuously up and down (a **wobble/siren alarm**). A sweep requires using variables in loops to increment the pitch (e.g. 500 -> 501 -> 502 -> ...). Since loops are not supported in interpreted mode, alternating between two distinct frequencies is the best way to simulate a high-visibility alert sound.

## Try This! (Challenges)
1. **Faster Siren**: Reduce the delays to `100` ms. Notice how the alarm sounds more urgent.
2. **Three-Tone Siren**: Add a third tone of `1000` Hz in the middle to create a cascading alarm sound.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The pitch does not change | Second tone call is missing | Ensure both `tone(BUZZ_PIN, 500)` and `tone(BUZZ_PIN, 1500)` are present with delays after each. |
| Sound is very quiet or rasping | PWM interference | Ensure pin 8 is used. If sharing with other timers, some interference can occur. |

## Mode Notes
Alternating two static `tone()` calls with delays is supported by MbedO interpreted mode.

## Related Projects
- [07 - Buzzer Beep](07-buzzer-beep.md)
- [08 - Buzzer Melody](08-buzzer-melody.md)
- [31 - Motion Alarm](../intermediate/31-motion-alarm.md)
