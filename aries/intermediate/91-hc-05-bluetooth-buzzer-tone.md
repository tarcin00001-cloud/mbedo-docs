# 91 - HC-05 Bluetooth Buzzer Tone Player

Play specific musical notes wirelessly by sending character commands from a Bluetooth device to a buzzer connected to the ARIES v3 board.

## Goal
Learn how to parse character commands to map inputs to specific pitch frequencies, and control a buzzer over a wireless UART link without using loops inside C++ code.

## What You Will Build
An HC-05 Bluetooth module on UART2 (pins 16, 17) receives serial characters. A buzzer is connected to GPIO 14. Sending note characters ('C', 'D', 'E', 'F', 'G', 'A', 'B') plays the corresponding pitch. Sending 'S' silences the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| VEGA ARIES v3 Board | `aries` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| Passive Buzzer (or Active Buzzer) | `buzzer` | Yes | Yes (on GPIO 14) |

## Wiring
| Component | Component Pin | ARIES Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO 16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO 17 (TX2) | Yellow | Serial Data |
| Buzzer | Positive (+) | GPIO 14 | Blue | Audio signal line |
| Buzzer | Negative (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the buzzer positive lead directly to **GPIO 14** and negative to the common ground rail.

## Code
```cpp
// HC-05 Bluetooth Buzzer Tone Player
const int BUZZER_PIN = 14;

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600);
  
  pinMode(BUZZER_PIN, OUTPUT);
  noTone(BUZZER_PIN);
  
  Serial.println("Bluetooth Tone Player Active. Send note character...");
}

void loop() {
  if (Serial2.available() > 0) {
    char cmd = (char)Serial2.read();
    
    // Process character command
    switch (cmd) {
      case 'C':
      case 'c':
        tone(BUZZER_PIN, 262); // Play C4
        Serial.println("Play: C4");
        Serial2.println("Playing note: C4 (262 Hz)");
        break;
      case 'D':
      case 'd':
        tone(BUZZER_PIN, 294); // Play D4
        Serial.println("Play: D4");
        Serial2.println("Playing note: D4 (294 Hz)");
        break;
      case 'E':
      case 'e':
        tone(BUZZER_PIN, 330); // Play E4
        Serial.println("Play: E4");
        Serial2.println("Playing note: E4 (330 Hz)");
        break;
      case 'F':
      case 'f':
        tone(BUZZER_PIN, 349); // Play F4
        Serial.println("Play: F4");
        Serial2.println("Playing note: F4 (349 Hz)");
        break;
      case 'G':
      case 'g':
        tone(BUZZER_PIN, 392); // Play G4
        Serial.println("Play: G4");
        Serial2.println("Playing note: G4 (392 Hz)");
        break;
      case 'A':
      case 'a':
        tone(BUZZER_PIN, 440); // Play A4 (Tuning reference)
        Serial.println("Play: A4");
        Serial2.println("Playing note: A4 (440 Hz)");
        break;
      case 'B':
      case 'b':
        tone(BUZZER_PIN, 494); // Play B4
        Serial.println("Play: B4");
        Serial2.println("Playing note: B4 (494 Hz)");
        break;
      case 'S':
      case 's':
        noTone(BUZZER_PIN);    // Stop tone
        Serial.println("Silence");
        Serial2.println("Buzzer Off");
        break;
    }
  }
}
```

## What to Click in MbedO
1. Drag the **VEGA ARIES v3**, **HC-05 Module**, and **Buzzer** onto the canvas.
2. Wire HC-05 TXD/RXD to **GPIO 16/GPIO 17**. Connect Buzzer positive to **GPIO 14**.
3. Paste the code into the editor.
4. Select **Interpreted Mode** in the simulation dropdown.
5. Click **Run** to execute the project.
6. In the virtual Bluetooth text console, type `E` and press Enter to hear the E4 note. Type `S` to turn it off.

## Expected Output
Serial Monitor:
```
Bluetooth Tone Player Active. Send note character...
Play: E4
Play: G4
Silence
```

Bluetooth Terminal:
```
Playing note: E4 (330 Hz)
Playing note: G4 (392 Hz)
Buzzer Off
```

## Expected Canvas Behavior
* Sending note characters over the Bluetooth console causes the buzzer widget to vibrate/sound at the corresponding pitch.
* Sending the character 'S' immediately silences the buzzer widget.
* Character cases do not affect note triggers (e.g. both 'c' and 'C' work).

## Code Walkthrough
| Line | Check / Action |
| --- | --- |
| `char cmd = (char)Serial2.read()` | Reads one byte/character directly from the serial queue buffer. |
| `switch (cmd)` | Uses a switch-case lookup block to map single characters to notes. |
| `tone(BUZZER_PIN, 440)` | Generates a 440 Hz reference pitch (Standard A4 note) on GPIO 14. |
| `noTone(BUZZER_PIN)` | Stops tone generation on GPIO 14. |

## Hardware & Safety Concept
* **Asynchronous Single-Byte Processing**: Reading single characters (`char`) is far more memory efficient and less prone to buffer overflow issues than reading full strings. The code evaluates each character as it arrives without waiting for newline terminators, reducing latency in time-sensitive interactive audio devices.
* **Passive vs. Active Buzzers**: A passive buzzer requires a pulsing frequency signal (like that generated by `tone()`) to produce distinct musical notes. If an active buzzer is used, it contains its own internal oscillator and will sound only at its pre-determined fixed frequency when powered, ignoring the frequency parameters of `tone()`.

## Try This! (Challenges)
1. **Chime Sequence**: Map character 'M' to play a quick three-note startup melody automatically using short delays in a loop-free sequence.
2. **Auto-Timeout Silencer**: Implement a timeout check using `millis()` so the buzzer automatically silences if no command is received for 3 seconds.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound is a continuous flat pitch | Active buzzer used instead of passive | Verify if a passive buzzer is needed to vary frequencies. |
| Buzzer does not sound | Pin mapping conflict | Verify pin 14 is set as output; ensure noTone is not being called elsewhere. |
| Wrong notes are playing | Switch case falling through | Ensure `break;` is present at the end of every case statement block. |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [70 - Active Buzzer Chime Scale Generator](70-active-buzzer-chime-scale-generator.md)
- [89 - HC-05 Bluetooth Serial UART Link](89-hc-05-bluetooth-serial-uart.md)
- [90 - HC-05 Bluetooth LED Controller](90-hc-05-bluetooth-led-controller.md)
