# 91 - ESP32 HC-05 Bluetooth Buzzer Tone Player

Play specific musical notes wirelessly by sending character commands from a Bluetooth device to a passive buzzer.

## Goal
Learn how to parse character commands to map inputs to specific pitch frequencies, and control a piezo speaker over a wireless UART link.

## What You Will Build
An HC-05 Bluetooth module on UART2 (pins 16, 17) receives serial characters. A passive buzzer is connected to GPIO 15. Sending note characters ('C', 'D', 'E', 'F', 'G', 'A', 'B') plays the corresponding pitch. Sending 'S' silences the buzzer.

## Parts Needed
| Part | MbedO Component Type | Required in MbedO | Required on hardware |
| --- | --- | --- | --- |
| ESP32 DevKitC | `esp32` | Yes | Yes |
| HC-05 Bluetooth Module | `bluetooth_hc05` | Yes | Yes |
| Passive Buzzer (Piezo Speaker) | `buzzer` | Yes | Yes |

## Wiring

| Component | Component Pin | ESP32 Pin | Wire Colour | Notes |
| --- | --- | --- | --- | --- |
| HC-05 Module | VCC | 5V (Vin) | Red | Power |
| HC-05 Module | GND | GND | Black | Ground |
| HC-05 Module | TXD | GPIO16 (RX2) | Green | Serial Data |
| HC-05 Module | RXD | GPIO17 (TX2) | Yellow | Serial Data |
| Passive Buzzer | Positive (+) | GPIO15 | Blue | Audio signal line |
| Passive Buzzer | Negative (−) | GND | Black | Ground reference |

> **Wiring tip:** Connect the passive buzzer positive lead directly to **GPIO 15** and negative to the common ground rail.

## Code
```cpp
// HC-05 Bluetooth Buzzer Tone Player
#define RX2_PIN 16
#define TX2_PIN 17

const int BUZZER_PIN = 15;

void setup() {
  Serial.begin(115200);
  Serial2.begin(9600, SERIAL_8N1, RX2_PIN, TX2_PIN);
  
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
1. Drag **ESP32 DevKitC**, **HC-05 Module**, and **Buzzer** onto the canvas.
2. Wire HC-05 TXD/RXD to **GPIO16/GPIO17**. Connect Buzzer positive to **GPIO15**.
3. Paste the code and click **Run**.
4. In the virtual Bluetooth text console, type `E` and press Enter to hear the E4 note. Type `S` to turn it off.

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
* Sending note characters over the Bluetooth console causes the buzzer widget to vibrate at the corresponding pitch.
* Sending the character 'S' immediately silences the buzzer widget.
* Character cases do not affect note triggers (e.g. both 'c' and 'C' work).

## Code Walkthrough
| Line | What It Does |
| --- | --- |
| `char cmd = (char)Serial2.read()` | Reads one byte/character directly from the serial queue buffer. |
| `switch (cmd)` | Uses a fast switch-case lookup block to map single characters to notes. |
| `tone(BUZZER_PIN, 440)` | Generates a 440 Hz reference pitch (Standard A4 note). |
| `noTone(BUZZER_PIN)` | Stops tone generation on GPIO 15. |

## Hardware & Safety Concept: Asynchronous Single-Byte Processing
Reading single characters (`char`) is far more memory efficient and less prone to buffer overflow exploits than reading full strings. The code evaluates each character as it arrives without waiting for newline terminators, reducing latency in time-sensitive interactive audio devices.

## Try This! (Challenges)
1. **Melody Trigger**: Map character 'M' to play a short pre-programmed melody (e.g., a startup tune).
2. **Auto-Timeout Chime**: Automatically shut off the buzzer if no new note is received within 3 seconds.
3. **Note Duration Controller**: Parse commands like "C500" to play note C for 500 ms and then stop.

## Troubleshooting
| Symptom | Likely Cause | Fix |
| --- | --- | --- |
| Sound is a continuous flat pitch | Active buzzer used instead of passive | Make sure to use a passive buzzer module |
| Buzzer does not sound | SPI or hardware UART conflict | Verify pin 15 is set as output; ensure noTone is not being called in the loop elsewhere |
| Wrong notes are playing | Switch case falling through | Ensure `break;` is present at the end of every case statement block |

## Mode Notes
This project runs in MbedO **interpreted C++ mode**.

## Related Projects
- [70 - ESP32 Active Buzzer Chime Scale Generator](70-esp32-active-buzzer-chime-scale-generator.md)
- [89 - ESP32 HC-05 Bluetooth Serial UART Link](89-esp32-hc05-bluetooth-serial-uart-link.md)
- [90 - ESP32 Bluetooth LED Controller](90-esp32-hc05-bluetooth-led-controller-string-parsing.md)
