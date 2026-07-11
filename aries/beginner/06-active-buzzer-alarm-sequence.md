# 06 - Active Buzzer Alarm sequence

Generate a multi-pulse emergency alarm sequence using an active buzzer connected to the VEGA ARIES v3 board.

## Goal
Learn how to program complex temporal patterns using sequential logic and delay commands without relying on loop structures.

## What You Will Build
An active buzzer connected to pin `GPIO 14` is programmed to emit a triple-pulse warning sequence (three short beeps of 100 milliseconds separated by 100 milliseconds of silence, followed by a 1-second interval of silence).

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |
| Breadboard & Wires | `breadboard` | Yes | Yes |

## Wiring

| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO 14 | Blue | Digital control signal |
| Active Buzzer | GND (-) | GND | Black | Ground connection |
| Shared power | VCC / GND | 5V / 3V3 / GND | Red / Black | Shared power rails |

> **Wiring tip:** Connect a 100 Ω resistor in series with the buzzer's positive terminal to protect the digital pin from current spikes.

## Code
```cpp
// Active Buzzer Alarm Sequence - VEGA ARIES v3
const int BUZZER_PIN = 14;

void setup() {
  // Configure the buzzer pin as a digital output
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  // --- First Pulse ---
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  delay(100);
  
  // --- Second Pulse ---
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  delay(100);
  
  // --- Third Pulse ---
  digitalWrite(BUZZER_PIN, HIGH);
  delay(100);
  digitalWrite(BUZZER_PIN, LOW);
  
  // --- Long silence between cycles ---
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and the **Buzzer** onto the canvas.
2. Wire the Buzzer's positive terminal to **GPIO 14** and the negative terminal to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the Buzzer widget emitting a rapid triple-pulse beep followed by a pause.

## Expected Output
Serial Monitor:
```
System Initialized.
Alarm Sequence Started.
Beep 1
Beep 2
Beep 3
Cycle Complete. Resting...
```

## Expected Canvas Behavior
* The Buzzer widget displays sound waves in three quick bursts, goes silent for 1 second, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `delay(100)` | Used to time both the short beep duration and the brief silence between beeps. |
| `digitalWrite(..., LOW)` | Silences the buzzer. |
| `delay(1000)` | Creates the 1-second window of silence between successive alarm cycles. |

## Hardware & Safety Concept: Alarm Warning Signatures and Continuous Operation
* **Alarm Warning Signatures**: Human hearing is highly adaptive: flat continuous tones are quickly ignored (auditory fatigue). Modulating the sound into patterns (like a triple-pulse chirp) draws attention and is far more effective as an emergency warning signature.
* **Continuous Operation**: Active buzzers can draw continuous current when stuck HIGH (e.g. if the code crashes or hangs). Always ensure the default state is LOW, and include safety hardware overrides or watchdog timers where applicable on real machines.

## Try This! (Challenges)
1. **Faster Alarm**: Shorten the pulse delays to 50 milliseconds to create a high-urgency alarm pattern.
2. **Double Pulse Alarm**: Modify the code to generate a double-beep pattern instead of a triple-pulse.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click but no tone | Pulse duration too short | If delay is too short (e.g. < 10 ms), the physical buzzer diaphragm cannot oscillate fully. Keep pulses above 50 ms |
| Sound is continuous | Code stuck | Reset the board or verify that `delay()` calls are present after every `digitalWrite` state change |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [05 - Active Buzzer Beep](05-active-buzzer-beep.md)
- [07 - External LED Blink (pin-tied)](07-external-led-blink.md) (Next project)
