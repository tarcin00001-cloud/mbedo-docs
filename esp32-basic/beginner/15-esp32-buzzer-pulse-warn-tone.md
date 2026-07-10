# 15 - ESP32 Buzzer Pulse Warn Tone

Build a high-intensity pulsing warning alert using an active buzzer with variable pulse-frequency warning sequences.

## Goal
Learn how to implement high-speed oscillation loops, construct alarm intervals using `for` loops inside the main runtime, and control piezo sound signatures.

## What You Will Build
An active buzzer on GPIO 25 generates a fast-pulsing alarm tone (10 pulses of 50 ms ON / 50 ms OFF), followed by a 1-second pause, creating an emergency lockout sound.

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

> **Wiring tip:** Connect the buzzer's positive pin (+) to GPIO25. Connect the negative pin (-) to a shared GND rail on your breadboard.

## Code
```cpp
// Active Buzzer connected to GPIO 25
const int BUZZER_PIN = 25;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  digitalWrite(BUZZER_PIN, LOW);
  
  Serial.begin(115200);
  Serial.println("ESP32 Pulsing Alert Siren Ready.");
}

void loop() {
  Serial.println(">> ALARM ACTIVE <<");
  
  // Pulse the buzzer 10 times rapidly
  // 50ms ON, 50ms OFF creates a rapid machine gun warning tone
  for (int i = 0; i < 10; i++) {
    digitalWrite(BUZZER_PIN, HIGH);
    delay(50);
    digitalWrite(BUZZER_PIN, LOW);
    delay(50);
  }
  
  // Pause for 1 second before next alarm burst
  Serial.println("Monitoring Area...");
  delay(1000);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Connect Buzzer **VCC** to **GPIO25**.
3. Connect Buzzer **GND** to ESP32 **GND**.
4. Paste the C++ code into the editor.
5. Select interpreted C++ mode and click **Run**.
6. Observe the rapid sound wave visual feedback on the canvas.

## Expected Output
Serial Monitor:
```
ESP32 Pulsing Alert Siren Ready.
>> ALARM ACTIVE <<
Monitoring Area...
>> ALARM ACTIVE <<
```

## Expected Canvas Behavior
* The buzzer emits 10 very rapid visual pulses, pauses for 1 second, then repeats.
* The Serial Monitor prints logs at the start of the alarm burst and during the pause.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `for (int i = 0; i < 10; i++)` | Executes the enclosed block of code exactly 10 times. |
| `delay(50);` | Keeps the buzzer ON or OFF for 50 milliseconds, creating a fast 10 Hz pulse rate. |

## Hardware & Safety Concept: Sound Pressure Levels (dB) and Pulse Warnings
Continuous tones are easily ignored by the human brain after a few minutes (sensory adaptation). To make warning systems effective, safety devices utilize **pulsing alarms**. The sudden onset and termination of sound triggers the brain's orienting reflex, drawing attention immediately. Active buzzers draw very little current (around 10 mA), making them safe to power directly from an ESP32 GPIO pin without a transistor.

## Try This! (Challenges)
1. **Accelerating Pulsar**: Modify the loop so that the delay decreases on each pulse (e.g. starting at 100 ms and reducing to 20 ms), creating a chirping sound.
2. **Double Tone Indicator**: Alternate between 5 rapid pulses and 5 slow pulses.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer click is too slow or too fast | Loop boundary error | Check that the `for` loop variable starts at 0 and ends at 10, and that both delay calls inside the loop are set to 50 ms. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [05 - ESP32 Active Buzzer Beep Alert](05-esp32-active-buzzer-beep-alert.md)
- [06 - ESP32 Active Buzzer Beep Alarm Pattern](06-esp32-active-buzzer-beep-alarm-pattern.md)
