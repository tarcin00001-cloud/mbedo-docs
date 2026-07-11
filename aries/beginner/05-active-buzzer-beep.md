# 05 - Active Buzzer Beep

Emit a periodic warning beep using an external active buzzer connected to the VEGA ARIES v3 board.

## Goal
Learn how to interface active buzzers, configure external digital output pins, and generate warning sounds using simple delay cycles.

## What You Will Build
An external active buzzer connected to pin `GPIO 14` is toggled ON and OFF every 1000 milliseconds, producing a steady pulsing warning beep.

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
// Active Buzzer Beep - VEGA ARIES v3
const int BUZZER_PIN = 14;

void setup() {
  // Configure the buzzer pin as a digital output
  pinMode(BUZZER_PIN, OUTPUT);
}

void loop() {
  // Turn the buzzer ON (sound tone)
  digitalWrite(BUZZER_PIN, HIGH);
  delay(1000);
  
  // Turn the buzzer OFF (silent)
  digitalWrite(BUZZER_PIN, LOW);
  delay(1000);
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3** board and the **Buzzer** onto the canvas.
2. Wire the Buzzer's positive terminal to **GPIO 14** and the negative terminal to **GND**.
3. Paste the code into the editor.
4. Click **Run**.
5. Observe the Buzzer widget emitting sound pulses and the console printing status updates.

## Expected Output
Serial Monitor:
```
System Initialized.
Buzzer State: ON
Buzzer State: OFF
```

## Expected Canvas Behavior
* The Buzzer widget displays sound waves for 1 second, goes silent for 1 second, and repeats.

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `const int BUZZER_PIN = 14` | Sets a symbolic constant name for the active buzzer pin (GPIO 14). |
| `pinMode(BUZZER_PIN, OUTPUT)` | Defines pin 14 as a digital output. |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives the pin HIGH, allowing current to flow and activating the buzzer. |

## Hardware & Safety Concept: Active vs. Passive Buzzers and Direct Pin Loads
* **Active vs. Passive Buzzers**: Active buzzers contain an internal oscillator that generates a fixed-frequency tone (usually 2.5 kHz) as soon as DC voltage is applied. Passive buzzers lack this oscillator and require a fluctuating AC signal (PWM/tones) to make sound.
* **Direct Pin Loads**: Buzzers are inductive loads and can draw up to 30 mA of current. While direct connection is possible in simulation, on real hardware it is safer to drive the buzzer using a NPN transistor (e.g. BC547) as a switch to prevent overloading the microcontroller's GPIO pin.

## Try This! (Challenges)
1. **Rapid Chirp**: Shorten the delays to 100 milliseconds to create a rapid status chirp.
2. **Asymmetric Beep**: Set the ON delay to 100 milliseconds and the OFF delay to 2000 milliseconds to simulate a low-power warning indicator.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| No sound is emitted | Wrong pin wiring | Verify that the buzzer positive (+) wire is connected to GPIO 14, not GPIO 15 |
| Sound is continuous | Wired to power rail | Ensure the buzzer positive terminal is connected to the GPIO signal pin, not directly to the 5V/3.3V power rails |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [04 - RGB LED Color Mixing (alternating delays)](04-rgb-led-color-mixing.md)
- [06 - Active Buzzer Alarm sequence](06-active-buzzer-alarm-sequence.md) (Next project)
