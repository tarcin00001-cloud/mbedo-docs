# 05 - ESP32 Active Buzzer Beep Alert

Build an audible warning click alert using an external active buzzer.

## Goal
Learn how to interface an active buzzer with the ESP32, distinguish between active and passive buzzers, and control audible warning timers in C++.

## What You Will Build
An active buzzer connected to GPIO 25 emits a short warning click (ON for 100 ms, OFF for 1000 ms), creating a periodic warning heartbeat.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| Active Buzzer Module | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| Active Buzzer | VCC (+) | GPIO25 | Orange | Audio control signal pin |
| Active Buzzer | GND (-) | GND | Black | Ground return |

> **Wiring tip:** Active buzzers are polarity-sensitive. The positive pin is usually marked with a plus (+) symbol. Connect it to GPIO25. Connect the negative pin to GND.

## Code
```cpp
// Active Buzzer connected to GPIO 25
const int BUZZER_PIN = 25;

// Alert timings in milliseconds
const int BEEP_DURATION = 100; // 100 ms chirp
const int SILENT_INTERVAL = 1000; // 1 second pause

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(115200);
  Serial.println("ESP32 Active Buzzer Alert Online.");
}

void loop() {
  // Turn buzzer ON
  digitalWrite(BUZZER_PIN, HIGH);
  Serial.println("Buzzer: BEEP");
  delay(BEEP_DURATION);
  
  // Turn buzzer OFF
  digitalWrite(BUZZER_PIN, LOW);
  delay(SILENT_INTERVAL);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Connect Buzzer **VCC** (+) to ESP32 **GPIO25**.
3. Connect Buzzer **GND** (-) to ESP32 **GND**.
4. Paste the C++ code into the editor.
5. Select interpreted mode and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Active Buzzer Alert Online.
Buzzer: BEEP
Buzzer: BEEP
```

## Expected Canvas Behavior
* The buzzer icon emits visual sound concentric wave ripples for 100 ms every 1.1 seconds.
* The terminal logs `Buzzer: BEEP` on each pulse.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `const int BUZZER_PIN = 25;` | Identifies GPIO 25 as the audio driver output. |
| `digitalWrite(BUZZER_PIN, HIGH);` | Sends 3.3V to the active buzzer module, triggering its internal oscillator. |
| `digitalWrite(BUZZER_PIN, LOW);` | Cuts voltage to the buzzer, silencing it. |

## Hardware & Safety Concept: Active vs Passive Buzzers
- **Active Buzzers**: Contain an internal oscillator. When supplied with direct current (3.3V or 5V DC), they automatically vibrate at a fixed pre-determined pitch (typically 2.5 kHz). They are easy to code (`digitalWrite` HIGH/LOW) but cannot play music.
- **Passive Buzzers**: Do not contain an oscillator. They require an AC square wave signal (varying frequencies) to vibrate. They can play different musical notes but require PWM or the `tone()` function.

## Try This! (Challenges)
1. **Audible Countdown**: Modify the code to beep 5 times quickly, then pause for 3 seconds, simulating a timer alarm.
2. **Pulse Width Modulation**: Try to change the buzzer volume by writing analog values (`analogWrite(BUZZER_PIN, 128)`). Note: Active buzzers do not support pitch changes, but changing average voltage can alter sound texture.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer is silent but code is running | Reversed polarity | Swap the wires. The positive side of the buzzer (+) must connect to GPIO25, not GND. |
| Buzzer click is too quiet | Under-voltage | Some active buzzers require 5V to sound at full volume. Try connecting the buzzer VCC to the 5V (VBUS) pin of the ESP32 (requires a transistor switch circuit for GPIO control). |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [06 - ESP32 Active Buzzer Alarm Pattern](06-esp32-active-buzzer-beep-alarm-pattern.md)
- [15 - ESP32 Buzzer Pulse Warn Tone](15-esp32-buzzer-pulse-warn-tone.md)
