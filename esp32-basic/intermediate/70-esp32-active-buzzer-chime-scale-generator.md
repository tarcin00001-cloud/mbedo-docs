# 70 - ESP32 Buzzer Chime Scale Generator

Generate a musical scale (octave chime) using a passive buzzer (piezo speaker) and the ESP32's `ledcWriteTone` or `tone` library function.

## Goal
Understand the difference between active and passive buzzers, learn how to generate specific musical frequencies (pitches) using PWM, and play a standard C-major scale.

## What You Will Build
A passive buzzer is connected to GPIO 15. The code iterates through an array of musical frequencies (C4 to C5) and plays each note for 500 ms with a brief gap, producing an ascending and descending musical octave.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Passive Buzzer (Piezo Speaker) | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Passive Buzzer | Positive (+) | GPIO15 | Blue | PWM signal pin |
| Passive Buzzer | Negative (−) | GND | Black | Ground reference |

> **Wiring tip:** Passive buzzers do not have a built-in oscillator; they require a square-wave AC signal to vibrate the piezo disc and produce sound. This allows you to generate arbitrary pitches by varying the PWM frequency. Active buzzers, by contrast, only generate one fixed tone when powered.

## Code
```cpp
// Buzzer Chime Scale Generator
const int BUZZER_PIN = 15;

// Define note frequencies in Hz (C Major Scale: C4 to C5)
const int notes[] = {
  262, // C4
  294, // D4
  330, // E4
  349, // F4
  392, // G4
  440, // A4
  494, // B4
  523  // C5
};

const int numNotes = sizeof(notes) / sizeof(notes[0]);

// Arduino-ESP32 Core includes standard tone() and noTone() wrapper functions.
// These automatically allocate an LEDC channel internally.

void setup() {
  Serial.begin(115200);
  Serial.println("Buzzer Scale Generator ready.");
}

void loop() {
  // Ascending scale
  Serial.println("--- Ascending Scale ---");
  for (int i = 0; i < numNotes; i++) {
    int freq = notes[i];
    Serial.print("Playing: "); Serial.print(freq); Serial.println(" Hz");
    
    tone(BUZZER_PIN, freq);
    delay(400); // Note duration
    
    noTone(BUZZER_PIN);
    delay(50); // Pause between notes
  }
  
  delay(1000); // Wait before next cycle
  
  // Descending scale
  Serial.println("--- Descending Scale ---");
  for (int i = numNotes - 1; i >= 0; i--) {
    int freq = notes[i];
    Serial.print("Playing: "); Serial.print(freq); Serial.println(" Hz");
    
    tone(BUZZER_PIN, freq);
    delay(400);
    
    noTone(BUZZER_PIN);
    delay(50);
  }
  
  delay(3000); // 3 second pause before repeating
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** (passive type) onto the canvas.
2. Connect Buzzer positive to **GPIO15** and negative to **GND**.
3. Paste the code and click **Run**.
4. Listen to the ascending and descending chimes.

## Expected Output
Serial Monitor:
```
Buzzer Scale Generator ready.
--- Ascending Scale ---
Playing: 262 Hz
Playing: 294 Hz
Playing: 523 Hz
--- Descending Scale ---
Playing: 523 Hz
Playing: 494 Hz
Playing: 262 Hz
```

## Expected Canvas Behavior
* The buzzer widget flashes or oscillates to indicate frequency output.
* You hear a clear 8-note rising scale, followed by a 1-second pause, then an 8-note falling scale, and a 3-second pause.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, freq)` | Starts a PWM signal on GPIO 15 at the requested frequency (Hz). |
| `noTone(BUZZER_PIN)` | Stops the PWM signal, silencing the buzzer. |
| `delay(50)` | Inserts a brief silence between notes to distinguish them. |

## Hardware & Safety Concept: Piezoelectric Audio Generation
A passive buzzer contains a thin slice of piezoelectric ceramic glued to a metal diaphragm. Applying an alternating voltage (square wave) causes the ceramic to rapidly expand and contract, vibrating the metal diaphragm to create sound pressure waves in the air. The frequency of the electrical wave corresponds directly to the pitch of the sound. To prevent high inductive kickback from damaging the microcontroller pins on larger speakers, a small current-limiting resistor (e.g. 100 Ω) or transistor driver is recommended.

## Try This! (Challenges)
1. **Play a Melody**: Input the notes and durations for a simple song (e.g., "Twinkle Twinkle Little Star") into arrays and play it.
2. **Speed controller**: Connect a potentiometer to GPIO 34 and use it to adjust the tempo (note duration) dynamically.
3. **Arpeggio cycle**: Create a major chord arpeggio (C - E - G - C) and play it as a continuous loop chime.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer only clicks once | Using an active buzzer instead of passive | Swap to a passive buzzer module |
| Sound is very quiet | Resistor too high or low VCC | Check series resistor values or verify direct connection to GPIO |
| Noise sounds distorted or raspy | Incorrect PWM frequency settings | Stick to standard note frequencies between 100 Hz and 2000 Hz |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [05 - ESP32 Active Buzzer Beep Alert](../beginner/05-esp32-active-buzzer-beep-alert.md)
- [06 - ESP32 Active Buzzer Beep Alarm Pattern](../beginner/06-esp32-active-buzzer-beep-alarm-pattern.md)
- [91 - ESP32 Bluetooth Buzzer Tone Player](91-esp32-bluetooth-buzzer-tone-player.md)
