# 06 - ESP32 Active Buzzer Alarm Pattern

Build an audible emergency siren using an active buzzer programmed with a multi-pulse alarm sequence.

## Goal
Learn how to structure nested helper functions, construct complex timing patterns using consecutive digital write sequences, and develop warning sound patterns.

## What You Will Build
An active buzzer on GPIO 25 plays a triple-beep alarm sequence (three rapid 100 ms chirps separated by 100 ms silences), followed by a 1.5-second silent interval, simulating a security breach warning.

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

> **Wiring tip:** Positive (+) pin of the active buzzer connects to GPIO25. Negative pin connects to a shared breadboard ground rail.

## Code
```cpp
// Active Buzzer connected to GPIO 25
const int BUZZER_PIN = 25;

void setup() {
  pinMode(BUZZER_PIN, OUTPUT);
  Serial.begin(115200);
  Serial.println("ESP32 Alarm Siren System Online.");
}

// Helper function to play a single short alert chirp
void playChirp(int duration) {
  digitalWrite(BUZZER_PIN, HIGH);
  delay(duration);
  digitalWrite(BUZZER_PIN, LOW);
}

void loop() {
  Serial.println(">> ALARM CYCLE START <<");
  
  // Play three rapid warning chirps
  playChirp(100);
  delay(100); // Pause between chirps
  
  playChirp(100);
  delay(100);
  
  playChirp(100);
  
  // Long pause before repeating the cycle
  Serial.println("System Monitoring...");
  delay(1500);
}
```

## What to Click in MbedO
1. Drag **ESP32 DevKitC** and **Buzzer** onto the canvas.
2. Connect Buzzer **VCC** to **GPIO25**.
3. Connect Buzzer **GND** to ESP32 **GND**.
4. Paste the C++ code into the editor.
5. Select interpreted mode and click **Run**.

## Expected Output
Serial Monitor:
```
ESP32 Alarm Siren System Online.
>> ALARM CYCLE START <<
System Monitoring...
>> ALARM CYCLE START <<
```

## Expected Canvas Behavior
* The buzzer emits three rapid visual sound ripples, pauses for 1.5 seconds, then repeats.
* The Serial Monitor prints the status logs synchronously.

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `void playChirp(int duration)` | Declares a helper function that encapsulates the logic to turn on, delay, and turn off the buzzer. |
| `playChirp(100);` | Calls the helper function to run the buzzer for 100 milliseconds. |

## Hardware & Safety Concept: Encapsulating Timing Blocks
In structured coding, wrapping repetitive sequences (like the ON-OFF-delay steps of a buzzer chirp) inside a helper function is key. It makes the main `loop()` easier to read, allows you to change beep duration across the entire program in one place, and prevents spelling or copy-paste errors when building complex alarm profiles.

## Try This! (Challenges)
1. **Critical Slowing Alarm**: Create a warning alarm that starts with rapid beeps (100 ms delay) and progressively increases the delay to 800 ms, simulating a cooling down process.
2. **Panic Siren**: Modify the code to toggle the buzzer ON and OFF continuously at a 50 ms rate, creating a rapid panic alarm.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Buzzer only beeps once, then clicks | Insufficient delay | Verify that there is a delay call *after* the final chirp in the loop. Otherwise, the loop immediately restarts, and the first beep of the next cycle blends into the last beep. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [05 - ESP32 Active Buzzer Beep Alert](05-esp32-active-buzzer-beep-alert.md)
- [15 - ESP32 Buzzer Pulse Warn Tone](15-esp32-buzzer-pulse-warn-tone.md)
