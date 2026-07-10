# 42 - Pico Sound Sensor

Build a sound-activated switch that detects clap sounds to toggle an LED.

## Goal
Learn how to interface digital output microphones (sound sensors) with GPIO pins on the Raspberry Pi Pico and toggle states on sound impulses.

## What You Will Build
A clap-activated light toggle:
- **Sound Sensor (GP16)**: Reads `LOW` when a sudden loud noise (clap) exceeds the microphone threshold (active-LOW trigger).
- **Warning LED (GP15)**: Toggles ON and OFF with each detected clap.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `button` | Yes (represented by switch button) | Yes (or microphone module) |
| LED (Red) | `led` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V | Power supply |
| Sound Sensor | OUT (Digital) | GP16 | Sound trigger output |
| Sound Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Light switch |
| Red LED | Cathode | GND | Ground return via resistor |

## Code
```cpp
const int SOUND_PIN = 16;
const int LED_PIN   = 15;

bool ledState = false;

void setup() {
  pinMode(SOUND_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  digitalWrite(LED_PIN, LOW); // Start with light OFF
}

void loop() {
  int soundTriggered = digitalRead(SOUND_PIN);

  // Active-LOW check: sound comparator pulls OUT to GND on spikes
  if (soundTriggered == LOW) {
    ledState = !ledState; // Toggle LED state
    digitalWrite(LED_PIN, ledState ? HIGH : LOW);
    delay(300); // Prevent double-triggering on the same sound wave
  }

  delay(10);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Sound Sensor** (represented by switch button), and **Red LED** onto the canvas.
2. Connect Sound Sensor: **VCC** to **3.3V**, **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Paste code, select the interpreted mode, and click **Run**.
5. Simulate a loud clap by clicking the sound switch on canvas.

## Expected Output

Terminal:
```
Simulation active. Sound switch monitor online.
```

## Expected Canvas Behavior
| Sound Input | GP16 Input State | LED state (GP15) | Light Status |
| --- | --- | --- | --- |
| Quiet | HIGH | Unchanged | Static |
| Clap Pulse | LOW | Toggled (ON) | **Switched ON** |
| Quiet | HIGH | Unchanged | Static |
| Clap Pulse | LOW | Toggled (OFF) | **Switched OFF** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `ledState = !ledState` | Toggles the boolean tracking state between true and false. |
| `delay(300)` | Blanking delay to let the sound vibration decay, avoiding multiple triggers from a single clap. |

## Hardware & Safety Concept: Sound Comparator Circuitry
Sound sensor modules contain a condenser microphone cartridge connected to an operational amplifier. The analog output of the microphone is checked against a reference voltage set by a trimmer potentiometer. When a sound wave creates a peak above this reference level, the comparator (usually LM393) pulls the digital output pin LOW.

## Try This! (Challenges)
1. **Double Clap Switch**: Modify the code to toggle the LED only when two distinct claps are heard within 600 ms.
2. **Alert Siren**: Add a buzzer on GP14 and sound a brief tone on each clap.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Light toggles continuously when quiet | Sensitivity too high | Turn the small blue potentiometer screw on the physical sensor board clockwise to reduce sensitivity. |

## Mode Notes
This basic logical latching project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [26 - Pico Button Counter](26-pico-button-counter.md)
- [50 - Pico Sound Alarm](50-pico-sound-alarm.md)
