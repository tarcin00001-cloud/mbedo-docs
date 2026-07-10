# 43 - Knock Doorbell

Trigger a classic doorbell chime when a physical knock is detected on a door.

## Goal
Learn how to use an analog vibration sensor (Piezo Sensor) to trigger a timed doorbell sequence on a buzzer, with lockout safety to filter out reverberations.

## What You Will Build
When someone knocks on a surface where the piezo sensor is mounted, it registers a vibration spike. If it exceeds 100, the buzzer connected to pin D8 plays a "ding-dong" chime (E5 at 659 Hz for 300 ms, then C5 at 523 Hz for 500 ms) and locks out for 2 seconds.

**Why A0 and D8?** Pin A0 measures the fast physical vibrations. Pin D8 outputs the audio doorbell chime.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Arduino Uno | `arduino` | Yes | Yes |
| Piezo Sensor | `piezo_sensor` | Yes | Yes |
| Buzzer | `buzzer` | Yes | Yes |
| 1M ohm resistor | `resistor` | Optional | Yes (absorbs voltage spikes) |
| 100 ohm resistor | `resistor` | Optional | Yes (protects buzzer) |

## Wiring
| Component | Component Pin | Arduino Pin | Notes |
| --- | --- | --- | --- |
| Piezo Sensor | POS | A0 | Analog signal connection |
| Piezo Sensor | NEG | GND | Ground reference |
| Buzzer | Pin 1 (Positive) | D8 | Audio control line |
| Buzzer | Pin 2 (Negative) | GND | Ground reference |

> On physical hardware, a **1 Megaohm resistor** must be placed in parallel across the Piezo Sensor's terminals (POS to NEG) to damp large voltage spikes and protect the Arduino inputs.

## Code
```cpp
const int PIEZO_PIN = A0;
const int BUZZ_PIN  = 8;

const int KNOCK_THRESHOLD = 100;

void setup() {
  Serial.begin(9600);
  Serial.println("Knock Doorbell Ready");
}

void loop() {
  int vibration = analogRead(PIEZO_PIN);
  
  // Check if vibration level crosses the knock threshold
  if (vibration > KNOCK_THRESHOLD) {
    Serial.print("Knock detected! Vibration: ");
    Serial.println(vibration);
    
    // Play "Ding" (Note E5: 659 Hz)
    tone(BUZZ_PIN, 659);
    delay(300);
    noTone(BUZZ_PIN);
    delay(80); // Brief pause
    
    // Play "Dong" (Note C5: 523 Hz)
    tone(BUZZ_PIN, 523);
    delay(500);
    noTone(BUZZ_PIN);
    
    // Lockout delay to allow vibrations to settle and avoid repeat ringing
    delay(2000); 
    Serial.println("System armed. Waiting for knock...");
  }
  
  delay(50); // Small poll loop delay
}
```

## What to Click in MbedO
1. Drag **Arduino Uno**, **Piezo Sensor**, and **Buzzer** onto the canvas.
2. Connect Piezo **POS** to Arduino **A0** and **NEG** to Arduino **GND**.
3. Connect Buzzer **pin 1** to Arduino **D8** and **pin 2** to Arduino **GND**.
4. Paste the code into the editor.
5. Use the default Arduino interpreted runtime.
6. Click **Run**.
7. Open the **Terminal** tab in the bottom dock.
8. Double-click the Piezo Sensor, adjust the vibration slider above 100, and listen to the chime.

## Expected Output

Terminal:
```
Knock Doorbell Ready
Knock detected! Vibration: 180
System armed. Waiting for knock...
...
```

### Expected Canvas Behavior

| Tap Action | Pin A0 Reading | Condition check | Buzzer State (D8) |
| --- | --- | --- | --- |
| Quiet | < 100 | False | Silent |
| Knock (Tap > 100) | > 100 | True | Plays Ding-Dong chime |

The buzzer vibrates on the canvas to play the two notes, then goes silent for the lockout duration.

## Code Walkthrough

| Line / Statement | What It Does |
| --- | --- |
| `int vibration = analogRead(PIEZO_PIN)` | Reads the raw vibration voltage spike. |
| `if (vibration > KNOCK_THRESHOLD)` | Evaluates if the vibration represents a door knock. If true, the code triggers the chime sequence. |
| `delay(2000)` | Implements a lockout period. This ensures physical door vibrations die out completely before checking for a new knock, preventing double rings. |

## Hardware & Safety Concept: Piezoelectric Transducers
Piezoelectric materials generate electricity when deformed. When someone knocks on a door, the impact generates a mechanical shockwave. The piezo crystal converts this shockwave into a electrical voltage spike. Since the crystal is highly sensitive, a single knock creates a oscillating voltage wave that could trigger the code multiple times; hence, a software lockout delay is required.

## Try This! (Challenges)
1. **Melody Doorbell**: Change the doorbell notes to play a three-note arpeggio (C5 -> E5 -> G5).
2. **Visual indicator**: Wire an LED to D13 and make it flash in sync with the chime.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Chime triggers repeatedly on a single knock | Lockout delay is missing or too short | Ensure `delay(2000)` is present at the end of the `if` block. |
| No sound | Buzzer ground missing | Check Buzzer Pin 2 connects directly to GND. |

## Mode Notes
These patterns (vibration analog reads triggering sequenced chimes with lockout delays) are supported by MbedO interpreted mode.

## Related Projects
- [09 - Doorbell Tone](../beginner/09-doorbell-tone.md)
- [23 - Knock Detector](../beginner/23-knock-detector.md)
