# 70 - Active Buzzer Chime Scale Generator

Generate sequential chime and warning beep rhythms on an active buzzer using the VEGA ARIES v3 board.

## Goal
Learn how to control active buzzers, manage temporal rhythm sequences using state variables, and structure periodic chime generators without using loops inside C++ code.

## What You Will Build
An active buzzer alarm system that cycles through three distinct beep patterns (short beep, medium beep, and long beep) in sequence to create a distinct status chime.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | Positive (+) | GPIO 14 | Red | Buzzer control line |
| Active Buzzer | Negative (-) | GND | Black | Ground reference |

> **Wiring tip:** Active buzzers are polarized devices. Ensure the longer pin or positive (+) marked terminal connects to GPIO 14 and the other terminal connects to GND.

## Code
```cpp
const int BUZZER_PIN = 14;
int beepState = 0;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  // Generate chime rhythms by cycling beep states loop-free
  if (beepState == 0) {
    // Pattern 1: Rapid short beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(80);
    digitalWrite(BUZZER_PIN, LOW);
    delay(200);
    beepState = 1;
  } else if (beepState == 1) {
    // Pattern 2: Medium warning beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(200);
    digitalWrite(BUZZER_PIN, LOW);
    delay(400);
    beepState = 2;
  } else if (beepState == 2) {
    // Pattern 3: Long alarm beep
    digitalWrite(BUZZER_PIN, HIGH);
    delay(500);
    digitalWrite(BUZZER_PIN, LOW);
    delay(800);
    beepState = 0;
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** and **Active Buzzer** components onto the canvas.
2. Wire the Buzzer: **Positive (+)** to **GPIO 14**, and **Negative (-)** to **GND**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.

## Expected Output
Serial Monitor:
```
System Initialized.
Buzzer Rhythm: Short Beep
Buzzer Rhythm: Medium Beep
Buzzer Rhythm: Long Beep
```

## Expected Canvas Behavior
* The active buzzer sounds a short, medium, and long beep in sequence, pauses, and then repeats the pattern.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `pinMode(BUZZER_PIN, OUTPUT)` | Sets GPIO 14 as output. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives pin HIGH, powering the active buzzer's internal oscillator to produce sound. |
| `delay(80)` | Sets the duration of the short beep tone. |
| `beepState = 1` | Advances the state variable to trigger the next pattern. |

## Hardware & Safety Concept
* **Active vs. Passive Buzzers**: Active buzzers contain an internal oscillator circuit. Applying a DC voltage (HIGH) causes them to beep at a fixed pre-determined pitch (usually 2.3 kHz). Passive buzzers do not have an oscillator; they require a square-wave AC signal (using `tone()` or PWM) to vibrate their piezo element and generate sound.
* **Back EMF Protection**: Piezoelectric crystals behave like capacitors and can generate feedback voltage spikes when compressed or deactivated. For larger buzzers, adding a reverse diode or a small resistor in series is good practice to protect the SoC pin.

## Try This! (Challenges)
1. **Double Beep**: Modify pattern 1 to sound two rapid 50 ms beeps separated by 50 ms before transitioning to pattern 2.
2. **Button Triggered Chime**: Add a push button on GPIO 16. Modify the code so the chime scale plays only once when the button is clicked.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer makes no sound | Polarity reversed or wrong pin | Verify the positive buzzer lead connects to GPIO 14 and the negative lead connects to GND (swapping them prevents sound). |
| Buzzer clicks instead of beeping | Delay value too short | Ensure the ON delay is at least 30-50 ms to allow the internal oscillator time to start up. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [05 - Active Buzzer Beep](../beginner/05-active-buzzer-beep.md)
- [06 - Active Buzzer Alarm Sequence](../beginner/06-active-buzzer-alarm-sequence.md)
