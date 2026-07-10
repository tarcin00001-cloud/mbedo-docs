# 08 - Buzzer Melody

Play a short musical melody on a buzzer using a sequence of `tone()` calls.

## Goal
Learn how to program a musical scale by sequencing frequency values and note durations using simple, non-array structures.

## What You Will Build
A buzzer connected to pin D8 plays a standard C major scale (8 notes: C4 to C5) once per loop iteration, with a 1-second pause between repetitions.

**Why D8?** D8 acts as our audio signal line. The buzzer is powered entirely by the digital pulses generated on D8.

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
  Serial.println("Buzzer Melody ready");
}

void loop() {
  // C major scale notes: C4, D4, E4, F4, G4, A4, B4, C5
  
  // Note 1: C4 (262 Hz)
  tone(BUZZ_PIN, 262); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 2: D4 (294 Hz)
  tone(BUZZ_PIN, 294); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 3: E4 (330 Hz)
  tone(BUZZ_PIN, 330); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 4: F4 (349 Hz)
  tone(BUZZ_PIN, 349); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 5: G4 (392 Hz)
  tone(BUZZ_PIN, 392); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 6: A4 (440 Hz)
  tone(BUZZ_PIN, 440); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 7: B4 (494 Hz)
  tone(BUZZ_PIN, 494); delay(300); noTone(BUZZ_PIN); delay(50);
  // Note 8: C5 (523 Hz)
  tone(BUZZ_PIN, 523); delay(300); noTone(BUZZ_PIN); delay(50);

  Serial.println("Melody played");
  delay(1000); // Pause before repeating
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
Buzzer Melody ready
Melody played
Melody played
...
```

### Expected Canvas Behavior

| Step / Note | frequency (Hz) | Duration (ms) | Inter-Note Silence |
| --- | --- | --- | --- |
| 1. C4 | 262 | 300 | 50 ms |
| 2. D4 | 294 | 300 | 50 ms |
| 3. E4 | 330 | 300 | 50 ms |
| 4. F4 | 349 | 300 | 50 ms |
| 5. G4 | 392 | 300 | 50 ms |
| 6. A4 | 440 | 300 | 50 ms |
| 7. B4 | 494 | 300 | 50 ms |
| 8. C5 | 523 | 300 | 50 ms |

The canvas buzzer visualizes audio output in quick bursts.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `tone(BUZZ_PIN, 262)` | Plays a Middle C note (262 Hz) on pin D8. |
| `delay(300)` | Keeps the note playing for 300 ms. |
| `noTone(BUZZ_PIN)` | Stops the note. |
| `delay(50)` | Pauses for 50 ms. This silence gap is crucial; without it, consecutive notes blend together into a single sound. |

## Hardware & Safety Concept: Note Pitch and Audio Frequencies
Pitch is determined by the **frequency** of the sound wave (measured in Hertz, or cycles per second).
- Lower numbers (e.g. 262 Hz) have longer wavelengths and sound lower to human ears.
- Higher numbers (e.g. 523 Hz) vibrate faster and sound higher.
The `tone()` function generates a square wave of the exact frequency specified, causing the buzzer's ceramic disk to vibrate at that frequency.

## Try This! (Challenges)
1. **Change the Tempo**: Change all `delay(300)` values to `delay(150)` and see how the speed of the scale changes.
2. **Arpeggio Pattern**: Create a custom melody by rearranging the notes. Play C4 (262), E4 (330), G4 (392), and C5 (523) in a jumping sequence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| The notes sound muddy or run together | Missing inter-note pauses | Ensure the `noTone(BUZZ_PIN); delay(50);` line is present between every single `tone()` call. |
| The scale plays too slowly | Tone delay is set too high | Lower the delay after the `tone()` call (e.g. from 300 ms to 200 ms). |

## Mode Notes
Sequenced `tone()` and `noTone()` calls with fixed parameters are supported by MbedO interpreted mode. Writing a loop that uses dynamic array indexing would trigger an interpreter warning, which is why a flat, step-wise list of notes is used.

## Related Projects
- [07 - Buzzer Beep](07-buzzer-beep.md)
- [09 - Doorbell Tone](09-doorbell-tone.md)
- [10 - Alarm Oscillate](10-alarm-oscillate.md)
