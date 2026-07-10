# 07 - Buzzer Beep

Make a buzzer produce a single beep tone using `tone()` and `noTone()`.

## Goal
Learn how to use the built-in Arduino `tone()` and `noTone()` functions to generate sound frequencies on a digital output pin.

## What You Will Build
A piezo buzzer connected to pin D8 plays a 1000 Hz beep for 500 ms, pauses for 500 ms, and repeats the cycle indefinitely.

**Why D8?** Buzzers can run on any digital I/O pin. D8 is used to keep the wiring simple and distinct from LED output pins.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 100 ohm resistor | `resistor` | Optional | Yes (optional, protects buzzer) |

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
  Serial.println("Buzzer Beep ready");
}

void loop() {
  // Generate a 1000 Hz (1 kHz) frequency tone
  tone(BUZZ_PIN, 1000);
  Serial.println("BEEP");
  delay(500); // Sound duration
  
  // Stop generating the tone
  noTone(BUZZ_PIN);
  delay(500); // Silent duration
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
Buzzer Beep ready
BEEP
BEEP
...
```

### Expected Canvas Behavior

| Time (ms) | Output State | Tone Output |
| --- | --- | --- |
| 0 - 500 | `tone(8, 1000)` | 1000 Hz Tone (Active) |
| 500 - 1000 | `noTone(8)` | Silent (Inactive) |
| 1000+ | Cycles | Alternates ON/OFF |

The buzzer component on the canvas flashes/vibrates when active.

## Code Walkthrough

| Line | What It Does |
| --- | --- |
| `tone(BUZZ_PIN, 1000)` | Starts a square wave of 1000 Hz on D8 to vibrate the internal buzzer element. |
| `noTone(BUZZ_PIN)` | Stops the sound wave generation. Without this, the buzzer will make sound forever. |

## Hardware & Safety Concept: Piezo Buzzers
A **Piezoelectric Buzzer** contains a small ceramic disk bonded to a metal plate. When you apply an oscillating voltage, the ceramic bends, creating sound pressure waves. 
- **Active Buzzers** have internal oscillators and buzz whenever DC power is connected.
- **Passive Buzzers** (like the one used in MbedO) require an external AC/PWM wave to oscillate. Using the `tone()` function allows us to specify the exact pitch (frequency) of the sound.

## Try This! (Challenges)
1. **Change the Pitch**: Modify the frequency parameter in `tone()` from `1000` to `440` (lower pitch) and then to `3000` (higher chirp).
2. **Double-Beep Pattern**: Modify the loop code to play two short 100 ms beeps followed by a 1-second silence.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No sound is produced | Ground connection missing | Verify Buzzer Pin 2 is connected to a GND pin on the Arduino. |
| Continuous sound with no breaks | Missing `noTone` call | Make sure you call `noTone(BUZZ_PIN)` after the first delay, and check for a second delay after `noTone`. |
| Click sound instead of tone | Signal pin on Ground | Swap the buzzer wires: Pin 1 to D8, Pin 2 to GND. |

## Mode Notes
These patterns (`tone` and `noTone` calls) are supported by MbedO interpreted mode and are safe for this beginner project.

## Related Projects
- [08 - Buzzer Melody](08-buzzer-melody.md)
- [09 - Doorbell Tone](09-doorbell-tone.md)
- [10 - Alarm Oscillate](10-alarm-oscillate.md)
