# 91 - Pico Bluetooth Buzzer

Play dynamic buzzer warning chimes wirelessly by sending character commands over Bluetooth.

## Goal
Learn how to parse multiple character options received over serial ports to play varying tone alerts on passive buzzers.

## What You Will Build
A wireless chime controller:
- **HC-05 Bluetooth Module (GP0, GP1)**: Receives commands.
- **Passive Buzzer (GP14)**: Plays a high warning beep when command `'H'` is received, a low warning tone on `'L'`, and stops sound on `'S'`.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth` | Yes (represented by serial UART receiver) | Yes |
| Passive Buzzer | `buzzer` | Yes | Yes |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| HC-05 | VCC | 5V | Power supply |
| HC-05 | TXD | GP1 (RX) | Data to Pico |
| HC-05 | RXD | GP0 (TX) | Data from Pico (with divider) |
| HC-05 | GND | GND | Ground reference |
| Passive Buzzer | VCC (+) | GP14 | PWM sound channel |
| Passive Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int BUZZER_PIN = 14;

void setup() {
  Serial1.begin(9600);
  pinMode(BUZZER_PIN, OUTPUT);
  Serial1.println("Bluetooth Buzzer Player Active. Send H, L, or S");
}

void loop() {
  if (Serial1.available()) {
    char cmd = Serial1.read();

    if (cmd == 'H') {
      tone(BUZZER_PIN, 880); // High warning tone (880 Hz)
      Serial1.println("Tone: HIGH");
    } 
    else if (cmd == 'L') {
      tone(BUZZER_PIN, 220); // Low warning tone (220 Hz)
      Serial1.println("Tone: LOW");
    } 
    else if (cmd == 'S') {
      noTone(BUZZER_PIN);     // Stop tone output
      Serial1.println("Tone: STOP");
    }
  }
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **HC-05 Bluetooth Module**, and **Passive Buzzer** onto the canvas.
2. Connect HC-05: **TXD** to **GP1**, **RXD** to **GP0**. Connect Buzzer to **GP14**.
3. Paste code, select the interpreted mode, and click **Run**.
4. Type `'H'` (High tone), `'L'` (Low tone), or `'S'` (Stop) in the Bluetooth terminal window to control the buzzer.

## Expected Output

Terminal:
```
Bluetooth Buzzer Player Active. Send H, L, or S
Tone: HIGH
Tone: LOW
Tone: STOP
```

## Expected Canvas Behavior
| Bluetooth Command | Buzzer (GP14) State | Sound Frequency |
| --- | --- | --- |
| `'H'` | PWM Active | 880 Hz (High) |
| `'L'` | PWM Active | 220 Hz (Low) |
| `'S'` | LOW (Silent) | (Silent) |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `tone(BUZZER_PIN, 880)` | Starts a continuous square wave at 880 Hz on GP14 to drive the passive buzzer. |
| `noTone(BUZZER_PIN)` | Disables PWM generation on GP14, bringing the pin LOW to stop sound output. |

## Hardware & Safety Concept: Pin DC Bias
When turning off a buzzer, always use `noTone()` rather than writing the pin `HIGH` or `LOW` directly. If you stop sound by writing a digital `HIGH`, constant DC current continues to flow through the buzzer's low resistance coil to ground. This wastes battery power, generates heat in the coil, and can burn out the microcontroller output pin. `noTone()` shuts off the PWM timer and drops the pin output to 0V.

## Try This! (Challenges)
1. **Beep Pattern**: Add a command `'P'` that plays a repeating two-tone chirp sequence using delays.
2. **Alert indicator**: Connect an external Red LED to GP15 and flash it in sync with the high alert tone.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click once but no tone | Active buzzer used | Confirm you are using a **passive** buzzer. Active buzzers cannot produce high/low pitch changes on frequency requests. |

## Mode Notes
This basic serial communication project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [70 - Pico Buzzer Scale](70-pico-buzzer-scale.md)
- [90 - Pico Bluetooth LED](90-pico-bt-led.md)
