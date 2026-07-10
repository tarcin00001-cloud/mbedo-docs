# 50 - Pico Sound Alarm

Trigger visual and acoustic alarms when ambient sound levels exceed a threshold.

## Goal
Learn how to coordinate multiple output alerts (LED and Buzzer) from digital sensor triggers on the Raspberry Pi Pico.

## What You Will Build
An acoustic security intrusion alert:
- **Sound Sensor (GP16)**: Reads `LOW` when noise levels cross the microphone threshold.
- **Warning LED (GP15)**: Turns ON (Red warning) immediately during noise detection.
- **Active Buzzer (GP14)**: Emits a warning chirp in sync with the alarm LED.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| Raspberry Pi Pico | `pico` | Yes | Yes |
| Sound Sensor Module | `button` | Yes (represented by switch button) | Yes (or mic module) |
| LED (Red) | `led` | Yes | Yes |
| Active Buzzer | `buzzer` | Yes | Yes |
| 220-ohm Resistor | `resistor` | Optional | Yes (current-limiting) |

## Wiring
| Component | Component Pin | Pico Pin | Notes |
| --- | --- | --- | --- |
| Sound Sensor | VCC | 3.3V | Power supply |
| Sound Sensor | OUT (Digital) | GP16 | Microphone trigger |
| Sound Sensor | GND | GND | Ground reference |
| Red LED | Anode | GP15 | Safety alert indicator |
| Red LED | Cathode | GND | Ground return via resistor |
| Active Buzzer | VCC (+) | GP14 | Acoustic alert indicator |
| Active Buzzer | GND (-) | GND | Ground return |

## Code
```cpp
const int SOUND_PIN  = 16;
const int LED_PIN    = 15;
const int BUZZER_PIN = 14;

void setup() {
  pinMode(SOUND_PIN, INPUT);
  pinMode(LED_PIN, OUTPUT);
  pinMode(BUZZER_PIN, OUTPUT);

  // Initialize outputs OFF
  digitalWrite(LED_PIN, LOW);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  int noiseDetected = digitalRead(SOUND_PIN);

  // Active-LOW trigger: sound sensor pulls OUT to GND on spikes
  if (noiseDetected == LOW) {
    digitalWrite(LED_PIN, HIGH);   // Light ON
    digitalWrite(BUZZER_PIN, HIGH); // Alarm sound ON
    delay(500);                     // Keep alarm active for 0.5s
  } else {
    digitalWrite(LED_PIN, LOW);
    digitalWrite(BUZZER_PIN, LOW);  // Silence
  }

  delay(20);
}
```

## What to Click in MbedO
1. Drag the **Raspberry Pi Pico**, **Sound Sensor** (represented by switch button), **Red LED**, and **Active Buzzer** onto the canvas.
2. Connect Sound Sensor: **VCC** to **3.3V**, **OUT** to **GP16**, **GND** to **GND**.
3. Connect LED: **Anode** to **GP15**, **Cathode** to **GND**.
4. Connect Buzzer: **VCC** to **GP14**, **GND** to **GND**.
5. Paste code, select the interpreted mode, and click **Run**.
6. Click the sound switch on canvas to simulate a loud sound and trigger the alarm.

## Expected Output

Terminal:
```
Simulation active. Sound threshold alarm online.
```

## Expected Canvas Behavior
| Sound Input | GP16 Input State | LED state (GP15) | Buzzer (GP14) | System Status |
| --- | --- | --- | --- | --- |
| Quiet | HIGH | LOW | LOW | Armed |
| Noise Trigger | LOW | HIGH (ON) | HIGH (ON) | **ALARM TRIGGERED (0.5s)** |

## Code Walkthrough
| Expression | What It Does |
| --- | --- |
| `digitalWrite(BUZZER_PIN, HIGH)` | Drives GP14 HIGH to sound the active buzzer coil. |
| `delay(500)` | Holds the alarm indicators ON for 500 ms so that brief sound spikes are clearly noticed by humans. |

## Hardware & Safety Concept: Sensor Latency
Microcontrollers execute loops in microseconds, whereas physical sound events (like a window breaking or a door slamming) happen in milliseconds. Without holding the output alarm state (using software timers or latch delays like `delay(500)`), the warning indicators would turn ON and OFF so quickly that human operators might not see or hear them.

## Try This! (Challenges)
1. **Latching Guard**: Rewrite the code so the alarm stays active until a reset button (GP17) is clicked.
2. **Double Trigger**: Make the system wait for two consecutive sound spikes within 1 second before sounding the alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer sounds weak or scratchy | Loose connection | Ensure the buzzer GND pin is tied to a physical Pico ground return pin. |

## Mode Notes
This basic acoustic safety project runs locally and is supported by MbedO interpreted mode.

## Related Projects
- [05 - Pico Active Buzzer](05-pico-active-buzzer.md)
- [42 - Pico Sound Sensor](42-pico-sound-sensor.md)
